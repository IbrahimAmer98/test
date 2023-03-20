using System;
using System.Activities.Expressions;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using System.Web.UI.WebControls;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Azure.WebJobs.Host;
using Microsoft.Xrm.Sdk;
using Microsoft.Xrm.Sdk.Query;
using Presistence.DAL;
using UOP.AzureFunctions.StudentRetention.DAL;
using UOP.AzureFunctions.StudentRetention.Helpers;
using UOP.AzureFunctions.StudentRetention.Helpers.Connections;
using UOP.AzureFunctions.StudentRetention.Model;
using UOP.AzureFunctions.StudentRetention.Model.MongoDBModels;



namespace UOP.AzureFunctions.StudentRetention.Functions
{
    public static class StudentAction
    {
        private static DMongoDB _dMongoDB;
        private static DCache _dCache;
        private static DTerm _dTermTest;
        private static DStudentProgressAndRiskLevel _dStudentProgressAndRiskLevel; 
        private static TermData _currentTerm;
        private static TermData _nextTerm;
        private static double? _CGPA;
        private static Guid? _arDivision;
        private static Guid? _enDivision;
        private static bool matchingdecisionRecords;
        //private static string _lang = Thread.CurrentThread.CurrentCulture.ToString() == "ar" ? "ar" : "en";
        
        [FunctionName("StudentAction")]
        public static HttpResponseMessage Run([HttpTrigger(AuthorizationLevel.Function, "get", Route = null)]HttpRequestMessage req, TraceWriter log)
         
        {
            try
            {
                #region request parameters & context validation
                // validation request is null 
                if (req == null)
                {
                    log.Error($"Invalid request, {nameof(req)} trigger is null or empty...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, {nameof(req)} trigger is null ...") };
                }
                var _webApiHelper = new WebAPIHelper(log);
                var parameters = _webApiHelper.GetQueryParameters(req.GetQueryNameValuePairs()); // Retrieve request query parameters to a dictionary 

                // check if url contain parameter
                if (parameters.Count == 0)
                {
                    // ////_log.Error($"Invalid request, no parameter in request...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, no parameter in request...") };
                }

                // get parameter from url
                var relevantStudentID = parameters.ContainsKey("token") && parameters["token"].Count > 0 ? new Guid(parameters["token"].FirstOrDefault()) : (Guid?)null;
                var decisionTableSource = parameters.ContainsKey("source") && parameters["source"].Count > 0 ? parameters["source"].FirstOrDefault() : null;
                matchingdecisionRecords = parameters.ContainsKey("matchingdecisionRecords") && parameters["matchingdecisionRecords"].Count > 0 ? Convert.ToBoolean(parameters["matchingdecisionRecords"].FirstOrDefault()) : Convert.ToBoolean(null);

                // validation relevent student id
                if (relevantStudentID == null || relevantStudentID == Guid.Empty)
                {
                    log.Error($"Invalid request, {nameof(relevantStudentID)} is null or empty...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, {nameof(relevantStudentID)} trigger is null or empty...") };
                }
                // validation Source
                if (decisionTableSource == null)
                {
                    log.Error($"Invalid request, {nameof(decisionTableSource)} is null ...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, {nameof(decisionTableSource)} trigger is null ...") };
                }
                // validation matching decision Records
                if (matchingdecisionRecords == null)
                {
                    log.Error($"Invalid request, {nameof(matchingdecisionRecords)} is null ...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, {nameof(matchingdecisionRecords)} trigger is null ...") };
                }
                #endregion

                log.Info($"{nameof(StudentAction)} started executing for student with Id= {nameof(relevantStudentID)} ...");
                _dMongoDB = new DMongoDB(log); // Connection with mongo db
                _dCache = new DCache(log);
                    
                #region Preparing Guids
                //Preparing Guids
                // get current and future term
                BCache _bCache = new BCache(log);
                var xmlGuidsEnumerations = _bCache.GetGuids(new List<string> {
                    "new_registrationsubstatus",
                    "new_registrationstatus" ,
                    "new_proctorsubstatus" ,
                    "subject",
                    "new_registrationfinalgrade",
                    "new_coursetype",
                      "new_division"  });

                if (xmlGuidsEnumerations == null || xmlGuidsEnumerations.Count == 0)
                    throw new Exception("failed to retrieve guids");
                #endregion

                #region get student information from mongo db
                //get all document related with student id and convert to object as type student info
                StudentInfo currentStudent = _dMongoDB.GetDocumentsFromStudentInfoCollection(relevantStudentID.Value).FirstOrDefault<StudentInfo>();
                if (currentStudent == null)
                {
                    log.Error($"Student not found ...");
                    return new HttpResponseMessage { Content = new StringContent($"Student not found ...") };
                }
                #endregion

                #region Get current Term and future term
                //get current and furtuer turme relevent current student
                List<TermData> currentAndFutureTerms = GetCurrentAndFutureTerms(currentStudent, log);
                _currentTerm = currentAndFutureTerms[0];
                _nextTerm = currentAndFutureTerms[1];

                #endregion

                #region get case info from mongo db
                //get all document related with student id and convert to object as type student info
                Guid? CaseSubject = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "subject" && g.LookupInstance == "Leave of Absence")?.LookupGuid;
                if (CaseSubject == null || CaseSubject == Guid.Empty)
                    return new HttpResponseMessage { Content = new StringContent($"failed to retrieve 'CaseSubject' Guid ...") };
                Console.WriteLine("case is :" + CaseSubject);
                Console.Write("case is :" + CaseSubject);
                List<CaseInfo> CaseInfoList = _dMongoDB.GetDocumentsFromCaseInfoCollection(relevantStudentID.Value, CaseSubject, _currentTerm.TermID);
                if (CaseInfoList.Count == 0)
                {

                }
                #endregion
                #region get source from request
                log.Info($"{nameof(StudentAction)} started executing for Source with source = {nameof(decisionTableSource)} ...");
                PortalEnums.DecisionTableSource source = decisionTableSource == "100000000" ? PortalEnums.DecisionTableSource.mobileApp : decisionTableSource == "100000001" ? PortalEnums.DecisionTableSource.botAPI : decisionTableSource == "100000002" ? PortalEnums.DecisionTableSource.studentDashboardTopBanner : PortalEnums.DecisionTableSource._skip;
                #endregion
                #region get  CountryofResidencePickList of  currentStudent
                
                currentStudent.CountryofResidencePickList = _dCache.GetCountries().Where(c => c.CountryID.Equals(currentStudent.CountryOfResidence)).ToList().FirstOrDefault().CountryofResidencePickList;
                #endregion

                #region get decision record 
                List<DecisionData> decisionType = _dCache.GetDecisionType();
                if (decisionType == null)
                {
                    log.Error($"decision not found ...");
                    return new HttpResponseMessage { Content = new StringContent($"decision not found ...") };
                }
                #endregion

                #region get Test student
                _dTermTest = new DTerm(log);
                Account tempStudent = _dTermTest.TestId(currentStudent.CRMID.Value);
                currentStudent.PaymentStyle = (PortalEnums.PaymentStyle?)(tempStudent?.new_PaymentStyle?.Value);
                currentStudent.OrientationCourseCreationStatus= (PortalEnums.OrientationCourseCreationStatus?)(tempStudent?.new_OrientationCourseCreationStatus.Value);
                currentStudent.ProgramRequested=tempStudent?.New_ProgramrequestedId?.Id;
                currentStudent.StudentProgress=tempStudent?.new_StudentProgress?.Id;

                #endregion
                #region get programActual And programRequested record 
                List<ProgramData>programs = _dCache.GetPrograms().ToList();
                ProgramData programRequested = programs.Where(t => t.ProgramID.Equals(currentStudent.ProgramRequested)).ToList().FirstOrDefault();
                ProgramData programActual = programs.Where(t => t.ProgramID.Equals(currentStudent.Program)).ToList().FirstOrDefault();
                if (programActual == null)
                {
                    log.Error($"programActual not found ...");
                    return new HttpResponseMessage { Content = new StringContent($"programActual not found ...") };
                }
                #endregion
                #region get StudentProgressAndRiskLevel record 
                _dStudentProgressAndRiskLevel = new DStudentProgressAndRiskLevel(log);
                StudentProgressAndRiskLevelData studentProgressAndRiskLevel= new StudentProgressAndRiskLevelData();
                if (currentStudent.StudentProgress.HasValue)
                {
                    studentProgressAndRiskLevel = _dStudentProgressAndRiskLevel.GetAllStudentProgressAndRiskLevel(new[] { currentStudent.StudentProgress.Value }).FirstOrDefault();
                    if (programActual == null)
                    {
                        log.Error($"programActual not found ...");
                        return new HttpResponseMessage { Content = new StringContent($"programActual not found ...") };
                    }
                    studentProgressAndRiskLevel.InactiveCounter = Math.Max(studentProgressAndRiskLevel.InactivityCounterConsecutive.HasValue ? (sbyte)studentProgressAndRiskLevel.InactivityCounterConsecutive.Value : 0,
                        studentProgressAndRiskLevel.InactivityCounterAcademicYear.HasValue ? (sbyte)studentProgressAndRiskLevel.InactivityCounterAcademicYear.Value : 0);
                }
                List<StudentActionOutput> DecisionRecord = GetDecisionRecord(currentStudent, decisionType,_dCache,source, xmlGuidsEnumerations,_bCache, CaseInfoList, currentAndFutureTerms, programActual, studentProgressAndRiskLevel, programRequested, log);

                #endregion

                if (matchingdecisionRecords == true)
                {
                        return req.CreateResponse(HttpStatusCode.OK, DecisionRecord);
                }
                return req.CreateResponse(HttpStatusCode.OK, new List<StudentActionOutput>() { DecisionRecord.FirstOrDefault() });
            }
            catch (Exception ex)
            {
                log.Error($"An error occurred while executing {nameof(StudentAction)}. Error details: {ex.Message}");
                return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"An error occurred while executing {nameof(StudentAction)}.. Error details: {ex.Message}") };
            }
        }

        /// <summary>
        /// Gets decisions record outputs
        /// </summary>
        /// <param name="GetDecisionRecord">list of all decisions record</param>
        /// <returns></returns>
        private static List<StudentActionOutput> GetDecisionRecord(StudentInfo currentStudent, List<DecisionData> decisionRecord, DCache dCache, PortalEnums.DecisionTableSource source , List<XMLGuidsEnumerationData> xmlGuidsEnumerations, BCache _bCache, List<CaseInfo> CaseInfoList, List<TermData> currentAndFutureTerms,ProgramData programActual,StudentProgressAndRiskLevelData studentProgressAndRiskLevel,ProgramData programRequested, TraceWriter log)
        {
            try
            {
                // get current and future terms


                //get current student current,previous,next registrations  consuming azure function
                var studentPrevCurrentNextRegs = RetrieveCurrentStudentPrevCurrentNextRegistrations(currentStudent, log) ?? new List<Model.PrevCurrentNextTermRegistrationDocument>();
                var studentCurrentTermTrans = RetrieveStudentTransactionsByTerm(currentStudent, _nextTerm.TermID.Value, log) ?? new List<PrevCurrentNextTermTransactionDocument>();
                Guid? _registrationOpenStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationstatus" && g.LookupInstance == "Open")?.LookupGuid;
                Guid? _registrationCloseStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationstatus" && g.LookupInstance == "Closed")?.LookupGuid;
                Guid? _registrationRegisteredSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationsubstatus" && g.LookupInstance == "Registered")?.LookupGuid;
                Guid? _registrationfinalgradeWithdrawalId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationfinalgrade" && g.LookupInstance == "Withdrawal (W)")?.LookupGuid;
                Guid? _registrationfinalgradeDroppedlId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationfinalgrade" && g.LookupInstance == "Dropped (DR)")?.LookupGuid;
                Guid? _registrationgradedSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationsubstatus" && g.LookupInstance == "Graded")?.LookupGuid;
                _arDivision = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_division" && g.LookupInstance == "UoPeopleAR")?.LookupGuid;
                 _enDivision = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_division" && g.LookupInstance == "UoPeople")?.LookupGuid;

                #region check if current student birthday is today
                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.StudentId} , {nameof(currentStudent.DateOfBirth)} = {currentStudent.DateOfBirth}");
                var decisionIsBirthday = CheckIfTodayIsBirthday(currentStudent.DateOfBirth, currentStudent, log) ? PortalEnums.DecisionTableIsBirthday.Yes : PortalEnums.DecisionTableIsBirthday.No;
                #endregion 

                #region check if Is first day of term
                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.StudentId} , {nameof(currentStudent.DateOfBirth)} = {currentStudent.DateOfBirth}");
               var decisionIsFirstDayOfTerm = CheckIfIsFirstDayOfTerm(_currentTerm.startDate, _currentTerm, log) ? PortalEnums.IsFirstDayOfTerm.Yes : PortalEnums.IsFirstDayOfTerm.No;
                #endregion

                #region check current student term starting 
                // check current student term starting 
                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , term starting id = {"currentStudent.StartingTerm.TermID"}");
                var decisionTermStarting = (PortalEnums.DecisionTableTermStarting?)null;

                if (currentStudent.TermStarting != null && currentStudent.TermStarting != Guid.Empty)
                {
                    if (_currentTerm.TermID == currentStudent.TermStarting)
                        decisionTermStarting = PortalEnums.DecisionTableTermStarting.Current;

                    else if (currentAndFutureTerms.FirstOrDefault(t => t.TermID == currentStudent.TermStarting) != null)
                        decisionTermStarting = PortalEnums.DecisionTableTermStarting.Upcoming;

                    else
                        decisionTermStarting = PortalEnums.DecisionTableTermStarting.Passed;
                }
                #endregion

                #region get CGPA Value for current student
                // String _CGPAChangeStatus = GetCGPAChangeStatus(currentStudent, log);
                // var CGPAChangeStatus = _CGPAChangeStatus == "100000000" ? PortalEnums.CGPAchangeStatus.NoSignificantChange : _CGPAChangeStatus == "100000001" ? PortalEnums.CGPAchangeStatus.Increased : PortalEnums.CGPAchangeStatus.Decreased;
                _CGPA= GetCGPAValue(currentStudent, log);
                #endregion

                #region check if current student has open registrations or current term
                //check if current student has open registrations or current term
                if (_registrationOpenStatusId == null || _registrationOpenStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve registration 'Open' status Guid");
                //"Registered" registration sub status

                if (_registrationRegisteredSubStatusId == null || _registrationRegisteredSubStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve registration 'Registered' sub status Guid");
                var studentRegsForCurrentTerm = (from reg in studentPrevCurrentNextRegs
                                                 where reg.Term == _currentTerm.TermID
                                                 && reg.RegistrationStatus == _registrationOpenStatusId
                                                 && reg.RegistrationSubStatus == _registrationRegisteredSubStatusId
                                                 select reg).ToList();
                ////_log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {studentRegsForCurrentTerm.Count} registrations for current term");
                var decisionHasOpenRegsCurrentTerm = studentRegsForCurrentTerm.Count > 0 ? PortalEnums.DecisionHasOpenRegsCurrentTerm.Yes : PortalEnums.DecisionHasOpenRegsCurrentTerm.No;

                #endregion

                #region check if current student Has due now transactions for registrations
                var currentOpenTransForRegistrations = (from trans in studentCurrentTermTrans

                                                        join reg in studentPrevCurrentNextRegs
                                                        on trans.RegistrationId equals reg.CrmRegistrationId

                                                        where trans.Term == _currentTerm.TermID &&
                                                              trans.TransactionStatus == (int)PortalEnums.TransactionStatus.Open &&
                                                              trans.TransactionSubStatus == (int)PortalEnums.TransactionSubStatus.DueNow &&
                                                              trans.RegistrationId != null &&
                                                              trans.RegistrationId != Guid.Empty &&
                                                              reg.RegistrationStatus == _registrationOpenStatusId &&
                                                              reg.RegistrationSubStatus == _registrationRegisteredSubStatusId &&
                                                              reg.RegistrationCrmStatus == (int)PortalEnums.RegistraruionCRMStatus.Active

                                                        select trans).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {currentOpenTransForRegistrations.Count} due now transactions for registrations");

                var decisionHasDueNowTransForRegs = currentOpenTransForRegistrations.Count > 0 ?
                    PortalEnums.DecisionTable_HasDueNowTransactionsForRegistrations.Yes :
                    PortalEnums.DecisionTable_HasDueNowTransactionsForRegistrations.No;

                #endregion  

                #region check if current student Has Oer due transactions for registrations
                var currentOpenTransForRegistrationsOverDue = (from trans in studentCurrentTermTrans

                                                        join reg in studentPrevCurrentNextRegs
                                                        on trans.RegistrationId equals reg.CrmRegistrationId

                                                        where trans.Term == _currentTerm.TermID &&
                                                              trans.TransactionStatus == (int)PortalEnums.TransactionStatus.Open &&
                                                              trans.TransactionSubStatus == (int)PortalEnums.TransactionSubStatus.Overdue &&
                                                              trans.RegistrationId != null &&
                                                              trans.RegistrationId != Guid.Empty &&
                                                              reg.RegistrationStatus == _registrationOpenStatusId &&
                                                              reg.RegistrationSubStatus == _registrationRegisteredSubStatusId &&
                                                              reg.RegistrationCrmStatus == (int)PortalEnums.RegistraruionCRMStatus.Active

                                                        select trans).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {currentOpenTransForRegistrations.Count} due now transactions for registrations");

                var decisionHasOverDueTransForRegs = currentOpenTransForRegistrationsOverDue.Count > 0 ?
                    PortalEnums.HasOverDueTransaction.Yes :
                    PortalEnums.HasOverDueTransaction.No;

                #endregion

                #region check if current student has pending registrations for next term
                //check if current student has pending registrations for next term

                Guid? registrationRequestPendingSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationsubstatus" && g.LookupInstance == "Request pending")?.LookupGuid;

                if (registrationRequestPendingSubStatusId == null || registrationRequestPendingSubStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve registration 'Request pending' sub status Guid");

                var studentPendingRegsForNextTerm = (from reg in studentPrevCurrentNextRegs
                                                     where reg.Term == _nextTerm.TermID
                                                     && reg.RegistrationStatus == _registrationOpenStatusId
                                                     && reg.RegistrationSubStatus == registrationRequestPendingSubStatusId
                                                     select reg).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {studentPendingRegsForNextTerm.Count} pending registrations for next term");

                var decisionHasPendingRegsNextTerm = studentPendingRegsForNextTerm.Count > 0 ? PortalEnums.DecisionHasPendingRegsNextTerm.Yes : PortalEnums.DecisionHasPendingRegsNextTerm.No;

                #endregion

                #region check if current student has pending registrations and no proctor assigned for next term
                //check if current student has pending registrations and no proctor assigned for next term
                Guid? CourseTypeRequiredProctoredId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_coursetype" && g.LookupInstance == "Required- Proctored")?.LookupGuid;

                if (CourseTypeRequiredProctoredId == null || CourseTypeRequiredProctoredId == Guid.Empty)
                    throw new Exception("failed to retrieve course type 'Required - proctored' status Guid");

                //"No proctor assigned yet" proctor sub status
                Guid? proctorNotAssignedYetSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_proctorsubstatus" && g.LookupInstance == "No proctor assigned yet")?.LookupGuid;

                if (proctorNotAssignedYetSubStatusId == null || proctorNotAssignedYetSubStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve proctor 'No proctor assigned yet' sub status Guid");


                var studentNoAssignedProctorPendingRegsForNextTerm = (from reg in studentPrevCurrentNextRegs
                                                                      where reg.Term == _nextTerm.TermID
                                                                      && reg.RegistrationStatus == _registrationOpenStatusId
                                                                      && reg.RegistrationSubStatus == registrationRequestPendingSubStatusId
                                                                      && reg.CourseType == CourseTypeRequiredProctoredId
                                                                      && (reg.ProctorStatus == proctorNotAssignedYetSubStatusId || reg.ProctorStatus == null)
                                                                      select reg).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {studentNoAssignedProctorPendingRegsForNextTerm.Count} pending registrations with not assigned proctors for next term");

                var decisionHasPendingRegsAndNoAssignedProctorNextTerm = studentNoAssignedProctorPendingRegsForNextTerm.Count > 0 ? PortalEnums.DecisionHasPendingRegsAndNoProctorAssignedNextTerm.Yes : PortalEnums.DecisionHasPendingRegsAndNoProctorAssignedNextTerm.No;

                #endregion

                #region check if current student has processing registrations for next term
                //check if current student has processing registrations for next term

                //"Processing Request" registration sub status
                Guid? registrationProcessingRequestSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationsubstatus" && g.LookupInstance == "Processing Request")?.LookupGuid;

                if (registrationProcessingRequestSubStatusId == null || registrationProcessingRequestSubStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve registration 'Processing Request' sub status Guid");

                var studentProcessingRegistrationsForNextTerm = (from reg in studentPrevCurrentNextRegs
                                                                 where reg.Term == _nextTerm.TermID
                                                                 && reg.RegistrationStatus == _registrationOpenStatusId
                                                                 && (reg.RegistrationSubStatus == registrationProcessingRequestSubStatusId || reg.RegistrationSubStatus == _registrationRegisteredSubStatusId)
                                                                 select reg).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {studentProcessingRegistrationsForNextTerm.Count} processing registrations for next term");

                var decisionHasProcessingRegistrationsNextTerm = studentProcessingRegistrationsForNextTerm.Count > 0 ? PortalEnums.DecisionHasProcessingRegsNextTerm.Yes : PortalEnums.DecisionHasProcessingRegsNextTerm.No;
                #endregion

                #region check if current student has Number of remaining inactive terms – consecutive
                BTerm bTerm = new BTerm(log);
                string TA_CONSECUTIVE_RULE = "CounterOfInactiveConsecutiveTerms";
                string TA_ACADEMIC_YEAR_RULE = "CounterOfInactiveTermsInAcademicYear";
                var consecutiveRule = _bCache.GetRule(TA_CONSECUTIVE_RULE);
                var academicYearRule = _bCache.GetRule(TA_ACADEMIC_YEAR_RULE);

                var termActivity = bTerm.GetTermActivityByStudentId(currentStudent.CRMID)?
                    .Where(ta => ta.term?.TermID == _currentTerm.TermID)
                    .OrderByDescending(x => x.term.termNumber)
                    .FirstOrDefault();

                var ruleValueOfConsecutive = Int32.Parse(consecutiveRule.RuleValue?.ToString());
                var ruleValueOfAcademicYear = Int32.Parse(academicYearRule.RuleValue?.ToString());

                var taValueOfConsecutive = termActivity?.inactivityCounterConsecutive ?? 0;
                var taValueOfAcademicYear = termActivity?.inactivityCounterOnAcademicYear ?? 0;

                var decisionInactivityCounterConsecutive = ruleValueOfConsecutive - taValueOfConsecutive;
                var decisionInactivityCounterOnAcademicYear = ruleValueOfAcademicYear - taValueOfAcademicYear;

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID}, term-activity = {termActivity?.termActivityId}, " +
                    $"{nameof(ruleValueOfConsecutive)} = {ruleValueOfConsecutive}, {nameof(taValueOfConsecutive)} = {taValueOfConsecutive}, " +
                    $"{nameof(ruleValueOfAcademicYear)} = {ruleValueOfAcademicYear}, {nameof(taValueOfAcademicYear)} = {taValueOfAcademicYear}");
                #endregion

                #region check if current student has Approved LOA case – Current term
                var decisionHasApprovedLOAcaseCurrentTerm = CaseInfoList.Count > 0 ? PortalEnums.DecisionHasApprovedLOAcaseCurrentTerm.Yes : PortalEnums.DecisionHasApprovedLOAcaseCurrentTerm.No;
                #endregion

                #region check if current student Dropped or withdrawal from all courses
                //get all document related with student id and convert to object as type student info
                var prevCurrentNextTermRegistrationList = _dMongoDB.GetDocumentsFromRegistrationsPrevCurrentNextTermCollection(currentStudent.CRMID, _currentTerm.TermID).ToList<PrevCurrentNextTermRegistration>();
                bool status = false;
                if (prevCurrentNextTermRegistrationList.Count > 0)
                {
                    var prevCurrentNextTermRegistrationOpenStatus = (from reg in prevCurrentNextTermRegistrationList
                                                                     where reg.Term == _currentTerm.TermID
                                                                     && reg.RegistrationStatus == _registrationOpenStatusId
                                                                     select reg).ToList();

                    if (prevCurrentNextTermRegistrationOpenStatus.Count != 0)
                    {
                        status = false;
                    }
                    else
                    {
                        var prevCurrentNextTermRegistrationCloseStatus = (from reg in prevCurrentNextTermRegistrationList
                                                                          where reg.Term == _currentTerm.TermID
                                                                            && reg.RegistrationStatus == _registrationCloseStatusId
                                                                            && reg.RegistrationSubStatus == _registrationgradedSubStatusId
                                                                            && ((reg.FinalGrade == _registrationfinalgradeWithdrawalId) || (reg.FinalGrade == _registrationfinalgradeDroppedlId))
                                                                          select reg).ToList();
                        if (prevCurrentNextTermRegistrationCloseStatus.Count > 0)
                        {
                            status = true;
                        }
                        else
                        {
                            status = false;
                        }
                    }
                }
                else
                {

                }
                var decisionHasDroppedOrWithdrawalFromAllCourses = status == true ? PortalEnums.DecisionHasDroppedorWithDrawalFromallCourses.Yes : PortalEnums.DecisionHasDroppedorWithDrawalFromallCourses.No;
                #endregion


                #region Prepare Registration Period
                var currnetDate = DateTime.UtcNow;
                PortalEnums.RegistrationPeriod  RegistrationPeriod;
                if (currnetDate > currentStudent.DateToOpenEarlyRegistrationForNextTerm  && currnetDate < currentStudent.DateToCloseEarlyRegistrationForNextTerm )
                {
                    RegistrationPeriod = PortalEnums.RegistrationPeriod.DuringEarly;
                }
                else if (currnetDate > currentStudent.DateToOpenLateRegistrationForNextTerm && currnetDate < currentStudent.DateToCloseLateRegistrationForNextTerm)
                {
                    RegistrationPeriod = PortalEnums.RegistrationPeriod.DuringLate;
                }
                else if (currnetDate > currentStudent.DateToCloseEarlyRegistrationForNextTerm && currnetDate < currentStudent.DateToOpenLateRegistrationForNextTerm)
                {
                    RegistrationPeriod = PortalEnums.RegistrationPeriod.DuringEarlyandLate;
                }
                else
                {
                    RegistrationPeriod = PortalEnums.RegistrationPeriod._skip;
                }

                #endregion

                #region Prepare CourseCodes List 
                var CourseCodes = (from reg in studentPrevCurrentNextRegs                                               
                                         select reg.CourseCode).Distinct().ToList();


                #endregion
                //get current academic week
                AcademicWeekData currentAcademicWeek = _bCache.GetCurrentWeek(_currentTerm.TermID.Value);
                if (currentAcademicWeek == null)
                {
                    log.Error($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , failed to retrieve current academic week");
                    throw new Exception($"failed to retrieve current academic week");
                    
                }
                

               List<DecisionData> matchingPersonalMessageDecisions = (from dt in decisionRecord.Where(t => t.DecisionType.Equals((int)PortalEnums.DecisionType.SoecificMessages)).ToList()
                                                       where (dt.Division == null || dt.Division == currentStudent.Division)
                                                       && (dt.DuplicateAppInputStudentStatus == PortalEnums.StudentStatus._Skip || dt.DuplicateAppInputStudentStatus == currentStudent.StudentStatus)
                                                       && (dt.InputDegreeTypeFrom == PortalEnums.DegreeTypeDecisionTable.Skip || dt.InputDegreeTypeFrom == programRequested.InputDegreeTypeFrom)
                                                       && (dt.PaymentStyle == PortalEnums.PaymentStyle._skip || dt.PaymentStyle == currentStudent.PaymentStyle)
                                                     && (dt.OrientationCourseCreationStatus == PortalEnums.OrientationCourseCreationStatus._skip || dt.OrientationCourseCreationStatus == currentStudent.OrientationCourseCreationStatus)
                                                       && (dt.CountryofResidence.ToString().Contains(currentStudent.CountryofResidencePickList.ToString())
                                                        ||   dt.CountryofResidence == null   || dt.CountryofResidence.ToString().Contains(PortalEnums.CountryofResidencePickList._skip.ToString())     )
                                                      
                                                       && (dt.FinancialStatus == PortalEnums.StudentFinancialStatus._Skip || dt.FinancialStatus == currentStudent.FinancialStatus)
                                                       && (dt.CourseCode == null || CourseCodes.Contains(dt.CourseCode))
                                                       && (dt.RegistrationPeriod == PortalEnums.RegistrationPeriod._skip || dt.RegistrationPeriod == RegistrationPeriod)
                                                       && (dt.HasDueNowTransactionsForRegistrations == PortalEnums.DecisionTable_HasDueNowTransactionsForRegistrations._Skip || dt.HasDueNowTransactionsForRegistrations == decisionHasDueNowTransForRegs)
                                                       && (dt.HasOverDueTransaction == PortalEnums.HasOverDueTransaction._skip|| dt.HasOverDueTransaction == decisionHasOverDueTransForRegs)
                                                       && (dt.HasOpenRegsCurrentTerm == PortalEnums.DecisionHasOpenRegsCurrentTerm._Skip || dt.HasOpenRegsCurrentTerm == decisionHasOpenRegsCurrentTerm)
                                                       && (dt.HasPendingRegsNextTerm == PortalEnums.DecisionHasPendingRegsNextTerm._Skip || dt.HasPendingRegsNextTerm == decisionHasPendingRegsNextTerm)
                                                       && (dt.HasPendingRegsAndNoProctorAssignedNextTerm == PortalEnums.DecisionHasPendingRegsAndNoProctorAssignedNextTerm._Skip || dt.HasPendingRegsAndNoProctorAssignedNextTerm == decisionHasPendingRegsAndNoAssignedProctorNextTerm)
                                                       && (dt.HasProcessingRegsNextTerm == PortalEnums.DecisionHasProcessingRegsNextTerm._Skip || dt.HasProcessingRegsNextTerm == decisionHasProcessingRegistrationsNextTerm)
                                                       && (dt.HasApprovedLOAcaseCurrentTerm == PortalEnums.DecisionHasApprovedLOAcaseCurrentTerm._Skip || dt.HasApprovedLOAcaseCurrentTerm == decisionHasApprovedLOAcaseCurrentTerm)
                                                       && (dt.HasDroppedOrWithdrawalFromAllCourses == PortalEnums.DecisionHasDroppedorWithDrawalFromallCourses._Skip || dt.HasDroppedOrWithdrawalFromAllCourses == decisionHasDroppedOrWithdrawalFromAllCourses)
                                                       && (dt.IsBirthday == PortalEnums.DecisionTableIsBirthday._Skip || dt.IsBirthday == decisionIsBirthday)
                                                       && (dt.IsFirstDayOfTerm == PortalEnums.IsFirstDayOfTerm._Skip || dt.IsFirstDayOfTerm == decisionIsFirstDayOfTerm)
                                                       && (dt.InputTermStarting == PortalEnums.DecisionTableTermStarting._Skip || dt.InputTermStarting == decisionTermStarting)
                                                       && (dt.FoundationProgramGroup == PortalEnums.foundationProgramGroup._skip || dt.FoundationProgramGroup == programActual.FoundationProgramGroupFix)
                                                       && (dt.foundationType == PortalEnums.FoundationType.Skip || dt.foundationType == programActual.FoundationType)
                                                       && (dt.DegreeProgramGroupProgramRequested == PortalEnums.DegreeProgramGroupProgramRequested._skip || dt.DegreeProgramGroupProgramRequested == programRequested.DegreeProgramGroup)
                                                       && (dt.InputDegreeLevelFrom == PortalEnums.degreeLevelDecisionTable.Skip || dt.InputDegreeLevelFrom == programRequested.degreeLevel)
                                                       && (dt.programGroup == PortalEnums.ProgramGroup.Skip || dt.programGroup == programRequested.programGroup)
                                                       && (dt.InputMajorTo == PortalEnums.MajorDecisionTable.Skip || dt.InputMajorTo == programRequested.major)
                                                       && (dt.programType == PortalEnums.ProgramType._skip || dt.programType == programRequested.programType)
                                                       && (dt.DegreeProgramGroup == PortalEnums.DegreeProgramGroupProgramRequested._skip || dt.DegreeProgramGroup == programRequested.DegreeProgramGroup)
                                                       && (dt.NumberOfRemainingInactiveTerms_Consecutive == null || dt.NumberOfRemainingInactiveTerms_Consecutive == decisionInactivityCounterConsecutive)
                                                       && (dt.NumberOfRemainingInactiveTerms_AcademicYear == decisionInactivityCounterOnAcademicYear || dt.NumberOfRemainingInactiveTerms_AcademicYear == null)
                                                       && (dt.TermNumberInAcademicYear == _currentTerm.TermNumberInAcademicYear || dt.TermNumberInAcademicYear == null)
                                                       && (dt.InactiveCounter == studentProgressAndRiskLevel.InactiveCounter || dt.InactiveCounter == null)
                                                       && (dt.OnHoldCounter == studentProgressAndRiskLevel.OnHoldCounter || dt.OnHoldCounter == null)
                                                       && (dt.RiskLevel == PortalEnums.RiskLevel._skip || dt.RiskLevel == studentProgressAndRiskLevel.RiskLevel)
                                                       && (dt.RiskType == PortalEnums.RiskType._skip || dt.RiskType == studentProgressAndRiskLevel.RiskType)
                                                       && (dt.WeekFrom <= currentAcademicWeek.WeekNumber || dt.WeekFrom == null)
                                                       && (dt.WeekTo >= currentAcademicWeek.WeekNumber || dt.WeekTo == null)
                                                       && ((dt.StartDateToShow >= DateTime.UtcNow && dt.StartDateToShow <= DateTime.UtcNow.AddDays((double)dt.NumberofDaystoShow)) || dt.StartDateToShow == null)
                                                       && (_currentTerm.ShowDeadLineToPayForCourses.Value.AddDays(-(double)dt.NumberofDaystoshowbeforepaymentdeadline) >= DateTime.UtcNow  || dt.NumberofDaystoshowbeforepaymentdeadline == null)
                                                       // && (dt.CGPAchangeStatus == CGPAChangeStatus || dt.CGPAchangeStatus == null )
                                                       && (dt.Source == PortalEnums.DecisionTableSource._skip || dt.Source == source)
                                                       select dt).OrderBy(d => d.DecisionPriority).ToList();
                if (matchingPersonalMessageDecisions == null || matchingPersonalMessageDecisions.Count ==0)
                {
                    matchingPersonalMessageDecisions = (from dt in decisionRecord.Where(t => t.DecisionType.Equals((int)PortalEnums.DecisionType.StudentRetention)).ToList()
                                                        where (dt.Division == null || dt.Division == currentStudent.Division)
                                                           && (dt.DuplicateAppInputStudentStatus == PortalEnums.StudentStatus._Skip || dt.DuplicateAppInputStudentStatus == currentStudent.StudentStatus)
                                                           && (dt.FinancialStatus == PortalEnums.StudentFinancialStatus._Skip || dt.FinancialStatus == currentStudent.FinancialStatus)
                                                           && (dt.HasDueNowTransactionsForRegistrations == PortalEnums.DecisionTable_HasDueNowTransactionsForRegistrations._Skip || dt.HasDueNowTransactionsForRegistrations == decisionHasDueNowTransForRegs)
                                                           && (dt.HasOpenRegsCurrentTerm == PortalEnums.DecisionHasOpenRegsCurrentTerm._Skip || dt.HasOpenRegsCurrentTerm == decisionHasOpenRegsCurrentTerm)
                                                           && (dt.HasPendingRegsNextTerm == PortalEnums.DecisionHasPendingRegsNextTerm._Skip || dt.HasPendingRegsNextTerm == decisionHasPendingRegsNextTerm)
                                                           && (dt.HasPendingRegsAndNoProctorAssignedNextTerm == PortalEnums.DecisionHasPendingRegsAndNoProctorAssignedNextTerm._Skip || dt.HasPendingRegsAndNoProctorAssignedNextTerm == decisionHasPendingRegsAndNoAssignedProctorNextTerm)
                                                           && (dt.HasProcessingRegsNextTerm == PortalEnums.DecisionHasProcessingRegsNextTerm._Skip || dt.HasProcessingRegsNextTerm == decisionHasProcessingRegistrationsNextTerm)
                                                           && (dt.HasApprovedLOAcaseCurrentTerm == PortalEnums.DecisionHasApprovedLOAcaseCurrentTerm._Skip || dt.HasApprovedLOAcaseCurrentTerm == decisionHasApprovedLOAcaseCurrentTerm)
                                                           && (dt.HasDroppedOrWithdrawalFromAllCourses == PortalEnums.DecisionHasDroppedorWithDrawalFromallCourses._Skip || dt.HasDroppedOrWithdrawalFromAllCourses == decisionHasDroppedOrWithdrawalFromAllCourses)
                                                           && (dt.IsBirthday == PortalEnums.DecisionTableIsBirthday._Skip || dt.IsBirthday == decisionIsBirthday)
                                                           && (dt.InputTermStarting == PortalEnums.DecisionTableTermStarting._Skip || dt.InputTermStarting == decisionTermStarting)                                                          
                                                           && (dt.NumberOfRemainingInactiveTerms_Consecutive == null || dt.NumberOfRemainingInactiveTerms_Consecutive == decisionInactivityCounterConsecutive)
                                                           && (dt.NumberOfRemainingInactiveTerms_AcademicYear == decisionInactivityCounterOnAcademicYear || dt.NumberOfRemainingInactiveTerms_AcademicYear == null)
                                                           && (dt.TermNumberInAcademicYear == _currentTerm.TermNumberInAcademicYear || dt.TermNumberInAcademicYear == null)
                                                           && (dt.WeekFrom <= currentAcademicWeek.WeekNumber || dt.WeekFrom == null)
                                                           && (dt.WeekTo >= currentAcademicWeek.WeekNumber || dt.WeekTo == null)
                                                           // && (dt.CGPAchangeStatus == CGPAChangeStatus || dt.CGPAchangeStatus == null )
                                                           && (dt.Source == PortalEnums.DecisionTableSource._skip || dt.Source == source)
                                                           select dt).OrderBy(d => d.DecisionPriority).ToList();
                }
                //Get relevant top banner decisions for current student ordered by priority
             
                    log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {matchingPersonalMessageDecisions.Count} top banner decisions for this student");
                    return matchingPersonalMessageDecisions.Select(d => new StudentActionOutput
                    {
                        ActionTitle = (d.ActionTitle) ?? "",
                        ActionTitleAR = (d.ActionTitle_ar) ?? "",
                        ActionLink = d.ActionLink ?? "",
                        MessageText = (d.OutputMessageToShow != null) ? DynamicFieldsHandler(d.OutputMessageToShow,currentStudent) : "",
                        MessageTextAR = (d.OutputMessageToShow_AR != null) ? DynamicFieldsHandler(d.OutputMessageToShow_AR,currentStudent) : "",
                        MessageTitle = (d.MessageTitle != null) ? DynamicFieldsHandler(d.MessageTitle,currentStudent) : "",
                        MessageTitleAR = (d.MessageTitle_ar != null) ? DynamicFieldsHandler(d.MessageTitle_ar,currentStudent) : "",
                        Slug = (d.Slug) ?? "",
                        BackgroundImage = (d.BackgroundImage) ?? ""
                    }).ToList();
                
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , failed to retrieve Decision record");
                throw new Exception($"failed to retrieve records");
            }
        }

        // check if current student berthday is today 
        private static bool CheckIfTodayIsBirthday(DateTime? birthDate, StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                var currentDay = DateTime.Now.Day;
                var currentMonth = DateTime.Now.Month;


                return birthDate?.Day == currentDay && birthDate?.Month == currentMonth;
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(CheckIfTodayIsBirthday)}] - an error occured ,  student id = {currentStudent.CRMID} , ex : {ex}");
                throw;
            }
        }
        private static bool CheckIfIsFirstDayOfTerm(DateTime? startDate, TermData currentTerm, TraceWriter log)
        {
            try
            {
                var currentDay = DateTime.Now.Day;
                var currentMonth = DateTime.Now.Month;


                return startDate?.Day == currentDay && startDate?.Month == currentMonth;
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(CheckIfTodayIsBirthday)}] - an error occured ,  student id = {currentTerm.TermID} , ex : {ex}");
                throw;
            }
        }

        /// <summary>
        /// This method retrieves previous , current , next registrations for current student by consuming Azure function, in which it reads 
        /// the data from Mongo DB
        /// </summary>
        /// <returns></returns>
        private static List<Model.PrevCurrentNextTermRegistrationDocument> RetrieveCurrentStudentPrevCurrentNextRegistrations(StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                WebAPIHelper _apiHelper = new WebAPIHelper(log);
                var uriBaseAddress = Environment.GetEnvironmentVariable("registrationUrl");
                var uriPath = "api/RetrievePreviousCurrentNextTermRegistrations";
                var TOKEN_NAME = Environment.GetEnvironmentVariable("registrationUrlHeaderTokenName");
                var TOKEN = Environment.GetEnvironmentVariable("registrationUrlHeaderToken");
                var parameters = new Dictionary<string, string>
                {
                    { "StudentId" , currentStudent.CRMID.ToString()}
                };

                var headers = new Dictionary<string, string>();
                headers.Add(TOKEN_NAME, TOKEN);

                var studentPrevCurrentNextRegistrations = _apiHelper.Get<List<Model.PrevCurrentNextTermRegistrationDocument>>(uriBaseAddress, uriPath, parameters, null, headers);
                log.Info($"[{nameof(RetrieveCurrentStudentPrevCurrentNextRegistrations)}] - number of regs in {studentPrevCurrentNextRegistrations}  = {studentPrevCurrentNextRegistrations.Count} , student id = {currentStudent.StudentId}");
                return studentPrevCurrentNextRegistrations;
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(RetrieveCurrentStudentPrevCurrentNextRegistrations)}] - an error occured , student id = {currentStudent.StudentId} , ex : {ex}");
                throw;
            }
        }

        /// <summary>
        /// This method retrieves term transactions for current student by consuming Azure function, in which it reads 
        /// the data from Mongo DB
        /// </summary>
        /// <returns></returns>
        private static List<PrevCurrentNextTermTransactionDocument> RetrieveStudentTransactionsByTerm(StudentInfo currentStudent, Guid termId, TraceWriter log)
        {
            try
            {
                WebAPIHelper _apiHelper = new WebAPIHelper(log);
                var uriBaseAddress = Environment.GetEnvironmentVariable("transactionUrl");
                var uriPath = "api/RetrievePreviousCurrentNextTermTransactions";
                var TOKEN_NAME = Environment.GetEnvironmentVariable("transactionUrlHeaderTokenName");
                var TOKEN = Environment.GetEnvironmentVariable("transactionUrlHeaderToken");
                var parameters = new Dictionary<string, string>
                {
                    { "StudentId" , currentStudent.CRMID.ToString() }
                };
                    var headers = new Dictionary<string, string>
                {
                    { TOKEN_NAME ,TOKEN}
                };

                var studentPrevCurrentNextTransactions = _apiHelper.Get<List<PrevCurrentNextTermTransactionDocument>>(uriBaseAddress, uriPath, parameters, null, headers);
                log.Info($"[{nameof(RetrieveStudentTransactionsByTerm)}] - number of trans in {nameof(studentPrevCurrentNextTransactions)}  = {studentPrevCurrentNextTransactions.Count} , student id = {currentStudent.StudentId}, term Id = {termId}");
                return studentPrevCurrentNextTransactions;
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(RetrieveStudentTransactionsByTerm)}] - an error occured when get Transactions By Term for , student id = {currentStudent.StudentId}, term id = {termId}, ex : {ex}");
                throw;
            }
            
        }

        /// <summary>
        /// This method retrieves current and future terms 
        /// the data from cash
        /// </summary>
        /// <returns></returns>
        private static List<TermData> GetCurrentAndFutureTerms(StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                // get current and future term
                BCache _bCache = new BCache(log);
                List<TermData> _currentAndFutureTerms = _bCache.GetCurrentAndFutureTerms();
                if (_currentAndFutureTerms == null || _currentAndFutureTerms.Count < 2)
                {
                    log.Error($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , failed to retrieve current & future terms");
                    throw new Exception($"failed to retrieve current term");
                }
                else
                {
                    return _currentAndFutureTerms;
                }

            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , failed to retrieve current & future terms");
                throw new Exception($"failed to retrieve current term");

            }
        }

        /// <summary>
        /// This method retrieves CGBA 
        /// the data from new term statistics
        /// </summary>
        /// <returns></returns>
        /// 
        private static String GetCGPAChangeStatus(StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                UOPeopleCRMConnection _crmConnection = new UOPeopleCRMConnection(log);
                XrmServiceContext _service = _crmConnection.getserviceProvider();
                using (var provider = _crmConnection.getserviceProvider())
                {
                    var term = (from a in provider.New_timeperiodSet
                                join c in provider.New_termstatisticsSet
                                on a.New_timeperiodId.Value equals c.new_termid.Id
                                where (c.new_studentid.Id == new Guid("134d10f3-4830-e911-a976-000d3a1998fb"))
                                orderby a.New_TermNumber descending
                                select new
                                {
                                    New_TermNumber = a.New_TermNumber,
                                    CGPAchangeStatus =c.new_CgpaChangeStatus
                                }
                               ).ToList().FirstOrDefault();
                    return term.CGPAchangeStatus.Value.ToString();
                }
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(GetCGPAChangeStatus)}] - student id = {currentStudent.CRMID} , failed to retrieve CGPA change status current terms");
                throw new Exception($"failed to retrieve CGPA change status current term");
            }
        }

        private static double? GetCGPAValue(StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                UOPeopleCRMConnection _crmConnection = new UOPeopleCRMConnection(log);
                XrmServiceContext _service = _crmConnection.getserviceProvider();
                using (var provider = _crmConnection.getserviceProvider())
                {
                    var term = (from a in provider.New_timeperiodSet
                                join c in provider.New_termstatisticsSet
                                on a.New_timeperiodId.Value equals c.new_termid.Id
                                where (c.new_studentid.Id == currentStudent.CRMID.Value)
                                orderby a.New_TermNumber descending
                                select new
                                {
                                    New_TermNumber = a.New_TermNumber,
                                    CGPAchangeStatus = c.new_CgpaChangeStatus,
                                    CGPA=c.New_CGPA
                                }
                               ).ToList().FirstOrDefault();
                    if( term != null )
                    {

                        if (term.CGPA.HasValue)
                        {
                            return  (double)term.CGPA.Value;
                        }
                        else
                        {
                            return  null;
                        }
                    }
                    else
                    {
                        return null;
                    }
                }
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(GetCGPAChangeStatus)}] - student id = {currentStudent.CRMID} , failed to retrieve CGPA change status current terms");
                throw new Exception($"failed to retrieve CGPA change status current term");
            }
        }
        private static string DynamicFieldsHandler (string inputString, StudentInfo currentStudent)
        {

            string studentNamePlaceholder = "Student_Name";
            string studentNameValue =null;
            string studentPreferredNamePlaceholder = "Student_Preferred_Name";
            string studentPreferredNameValue = null; ;
            string studentCountryPlaceholder = "Student_Country";
            string studentCountryValue = null; ;
            string programPlaceholder = "Program";
            string programValue = null; ;
            string degreePlaceholder = "Degree";
            string degreeValue = null; ;
            if (currentStudent.Division.Value != null && currentStudent.Division.Value != Guid.Empty)
            {
            if(currentStudent.Division.Value == _arDivision)
                {
                    studentNameValue = "John Doe";
                    studentPreferredNameValue = "John Doe";
                    studentCountryValue = "John Doe";
                    programValue = "John Doe";
                    degreeValue = "John Doe";
                }  
            else if(currentStudent.Division.Value == _enDivision)

                {
                    studentNameValue = "John Doe";
                    studentPreferredNameValue = "John Doe";
                    studentCountryValue = "John Doe";
                    programValue = "John Doe";
                    degreeValue = "John Doe";
                }
                }
            
            string firstDayOfTheUpcomingTermPlaceholder = "First_Day_Of_The_Upcoming_Term";
            string firstDayOfTheUpcomingTermValue = _nextTerm?.startDate.Value.ToString();
            string lastDayOfTheUpcomingTermPlaceholder = "Last_Day_Of_The_Upcoming_Term";
            string lastDayOfTheUpcomingTermValue = _nextTerm?.endDate.Value.ToString();
            string registrationPeriodDatesPlaceholder = "Registration_Period_Dates";
            string registrationPeriodDatesValue = "Early=";
            string lateRegistrationPeriodDatesPlaceholder = "Late_Registration_Period_Dates";
            string lateRegistrationPeriodDatesValue = "John Doe";
            string studentsRegistrationPeriodDatesPlaceholder = "Students_Registration_Period_Dates";
            string studentsRegistrationPeriodDatesValue = "John Doe";
            string paymentDeadlinePlaceholder = "Payment_Deadline";
            string paymentDeadlineValue = _currentTerm?.ShowDeadLineToPayForCourses.Value.ToString();
            string CGPAPlaceholder = "CGPA";
            string CGPAValue = _CGPA.ToString();
            string maxCoursesAllowedPlaceholder = "Max_Courses_Allowed";
            string maxCoursesAllowedValue = currentStudent?.CourseLoadAllowed.Value.ToString();
            string orientationDeadlinePlaceholder = "Orientation_Deadline";
            string orientationDeadlineValue = "John Doe";
            ; List<object> replacements = new List<object>()
                                      {
                                     $"{studentNamePlaceholder}:{studentNameValue}",
                                     $"{studentPreferredNamePlaceholder}:{studentPreferredNameValue}",
                                     $"{studentCountryPlaceholder}:{studentCountryValue}",
                                     $"{programPlaceholder}:{programValue}",
                                     $"{degreePlaceholder}:{degreeValue}",
                                     $"{firstDayOfTheUpcomingTermPlaceholder}:{firstDayOfTheUpcomingTermValue}",
                                     $"{lastDayOfTheUpcomingTermPlaceholder}:{lastDayOfTheUpcomingTermValue}",
                                     $"{registrationPeriodDatesPlaceholder}:{registrationPeriodDatesValue}",
                                     $"{lateRegistrationPeriodDatesPlaceholder}:{lateRegistrationPeriodDatesValue}",
                                     $"{studentsRegistrationPeriodDatesPlaceholder}:{studentsRegistrationPeriodDatesValue}",
                                     $"{paymentDeadlinePlaceholder}:{paymentDeadlineValue}",
                                     $"{CGPAPlaceholder}:{CGPAValue}",
                                     $"{maxCoursesAllowedPlaceholder}:{maxCoursesAllowedValue}",
                                     $"{orientationDeadlinePlaceholder}:{orientationDeadlineValue}",
                                  
                                    };

            return ReplacePlaceholders(inputString, replacements);
        }
        private static string ReplacePlaceholders(string inputString, List<object> replacements)
        {
            foreach (object replacement in replacements)
            {
                string placeholder = "#" + replacement.ToString().Split(':')[0] + "#";
                string value = replacement.ToString().Split(':')[1]; inputString = inputString.Replace(placeholder, value);
            }
            return inputString;
        }



    }
}
using System;
using System.Activities.Expressions;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using System.Web.UI.WebControls;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Azure.WebJobs.Host;
using Microsoft.Xrm.Sdk;
using Microsoft.Xrm.Sdk.Query;
using Presistence.DAL;
using UOP.AzureFunctions.StudentRetention.DAL;
using UOP.AzureFunctions.StudentRetention.Helpers;
using UOP.AzureFunctions.StudentRetention.Helpers.Connections;
using UOP.AzureFunctions.StudentRetention.Model;
using UOP.AzureFunctions.StudentRetention.Model.MongoDBModels;



namespace UOP.AzureFunctions.StudentRetention.Functions
{
    public static class StudentAction
    {
        private static DMongoDB _dMongoDB;
        private static DCache _dCache;
        private static DTerm _dTermTest;
        private static DStudentProgressAndRiskLevel _dStudentProgressAndRiskLevel; 
        private static TermData _currentTerm;
        private static TermData _nextTerm;
        private static double? _CGPA;
        private static Guid? _arDivision;
        private static Guid? _enDivision;
        private static bool matchingdecisionRecords;
        //private static string _lang = Thread.CurrentThread.CurrentCulture.ToString() == "ar" ? "ar" : "en";
        
        [FunctionName("StudentAction")]
        public static HttpResponseMessage Run([HttpTrigger(AuthorizationLevel.Function, "get", Route = null)]HttpRequestMessage req, TraceWriter log)
         
        {
            try
            {
                #region request parameters & context validation
                // validation request is null 
                if (req == null)
                {
                    log.Error($"Invalid request, {nameof(req)} trigger is null or empty...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, {nameof(req)} trigger is null ...") };
                }
                var _webApiHelper = new WebAPIHelper(log);
                var parameters = _webApiHelper.GetQueryParameters(req.GetQueryNameValuePairs()); // Retrieve request query parameters to a dictionary 

                // check if url contain parameter
                if (parameters.Count == 0)
                {
                    // ////_log.Error($"Invalid request, no parameter in request...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, no parameter in request...") };
                }

                // get parameter from url
                var relevantStudentID = parameters.ContainsKey("token") && parameters["token"].Count > 0 ? new Guid(parameters["token"].FirstOrDefault()) : (Guid?)null;
                var decisionTableSource = parameters.ContainsKey("source") && parameters["source"].Count > 0 ? parameters["source"].FirstOrDefault() : null;
                matchingdecisionRecords = parameters.ContainsKey("matchingdecisionRecords") && parameters["matchingdecisionRecords"].Count > 0 ? Convert.ToBoolean(parameters["matchingdecisionRecords"].FirstOrDefault()) : Convert.ToBoolean(null);

                // validation relevent student id
                if (relevantStudentID == null || relevantStudentID == Guid.Empty)
                {
                    log.Error($"Invalid request, {nameof(relevantStudentID)} is null or empty...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, {nameof(relevantStudentID)} trigger is null or empty...") };
                }
                // validation Source
                if (decisionTableSource == null)
                {
                    log.Error($"Invalid request, {nameof(decisionTableSource)} is null ...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, {nameof(decisionTableSource)} trigger is null ...") };
                }
                // validation matching decision Records
                if (matchingdecisionRecords == null)
                {
                    log.Error($"Invalid request, {nameof(matchingdecisionRecords)} is null ...");
                    return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"Invalid request, {nameof(matchingdecisionRecords)} trigger is null ...") };
                }
                #endregion

                log.Info($"{nameof(StudentAction)} started executing for student with Id= {nameof(relevantStudentID)} ...");
                _dMongoDB = new DMongoDB(log); // Connection with mongo db
                _dCache = new DCache(log);
                    
                #region Preparing Guids
                //Preparing Guids
                // get current and future term
                BCache _bCache = new BCache(log);
                var xmlGuidsEnumerations = _bCache.GetGuids(new List<string> {
                    "new_registrationsubstatus",
                    "new_registrationstatus" ,
                    "new_proctorsubstatus" ,
                    "subject",
                    "new_registrationfinalgrade",
                    "new_coursetype",
                      "new_division"  });

                if (xmlGuidsEnumerations == null || xmlGuidsEnumerations.Count == 0)
                    throw new Exception("failed to retrieve guids");
                #endregion

                #region get student information from mongo db
                //get all document related with student id and convert to object as type student info
                StudentInfo currentStudent = _dMongoDB.GetDocumentsFromStudentInfoCollection(relevantStudentID.Value).FirstOrDefault<StudentInfo>();
                if (currentStudent == null)
                {
                    log.Error($"Student not found ...");
                    return new HttpResponseMessage { Content = new StringContent($"Student not found ...") };
                }
                #endregion

                #region Get current Term and future term
                //get current and furtuer turme relevent current student
                List<TermData> currentAndFutureTerms = GetCurrentAndFutureTerms(currentStudent, log);
                _currentTerm = currentAndFutureTerms[0];
                _nextTerm = currentAndFutureTerms[1];

                #endregion

                #region get case info from mongo db
                //get all document related with student id and convert to object as type student info
                Guid? CaseSubject = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "subject" && g.LookupInstance == "Leave of Absence")?.LookupGuid;
                if (CaseSubject == null || CaseSubject == Guid.Empty)
                    return new HttpResponseMessage { Content = new StringContent($"failed to retrieve 'CaseSubject' Guid ...") };
                Console.WriteLine("case is :" + CaseSubject);
                Console.Write("case is :" + CaseSubject);
                List<CaseInfo> CaseInfoList = _dMongoDB.GetDocumentsFromCaseInfoCollection(relevantStudentID.Value, CaseSubject, _currentTerm.TermID);
                if (CaseInfoList.Count == 0)
                {

                }
                #endregion
                #region get source from request
                log.Info($"{nameof(StudentAction)} started executing for Source with source = {nameof(decisionTableSource)} ...");
                PortalEnums.DecisionTableSource source = decisionTableSource == "100000000" ? PortalEnums.DecisionTableSource.mobileApp : decisionTableSource == "100000001" ? PortalEnums.DecisionTableSource.botAPI : decisionTableSource == "100000002" ? PortalEnums.DecisionTableSource.studentDashboardTopBanner : PortalEnums.DecisionTableSource._skip;
                #endregion
                #region get  CountryofResidencePickList of  currentStudent
                
                currentStudent.CountryofResidencePickList = _dCache.GetCountries().Where(c => c.CountryID.Equals(currentStudent.CountryOfResidence)).ToList().FirstOrDefault().CountryofResidencePickList;
                #endregion

                #region get decision record 
                List<DecisionData> decisionType = _dCache.GetDecisionType();
                if (decisionType == null)
                {
                    log.Error($"decision not found ...");
                    return new HttpResponseMessage { Content = new StringContent($"decision not found ...") };
                }
                #endregion

                #region get Test student
                _dTermTest = new DTerm(log);
                Account tempStudent = _dTermTest.TestId(currentStudent.CRMID.Value);
                currentStudent.PaymentStyle = (PortalEnums.PaymentStyle?)(tempStudent?.new_PaymentStyle?.Value);
                currentStudent.OrientationCourseCreationStatus= (PortalEnums.OrientationCourseCreationStatus?)(tempStudent?.new_OrientationCourseCreationStatus.Value);
                currentStudent.ProgramRequested=tempStudent?.New_ProgramrequestedId?.Id;
                currentStudent.StudentProgress=tempStudent?.new_StudentProgress?.Id;

                #endregion
                #region get programActual And programRequested record 
                List<ProgramData>programs = _dCache.GetPrograms().ToList();
                ProgramData programRequested = programs.Where(t => t.ProgramID.Equals(currentStudent.ProgramRequested)).ToList().FirstOrDefault();
                ProgramData programActual = programs.Where(t => t.ProgramID.Equals(currentStudent.Program)).ToList().FirstOrDefault();
                if (programActual == null)
                {
                    log.Error($"programActual not found ...");
                    return new HttpResponseMessage { Content = new StringContent($"programActual not found ...") };
                }
                #endregion
                #region get StudentProgressAndRiskLevel record 
                _dStudentProgressAndRiskLevel = new DStudentProgressAndRiskLevel(log);
                StudentProgressAndRiskLevelData studentProgressAndRiskLevel= new StudentProgressAndRiskLevelData();
                if (currentStudent.StudentProgress.HasValue)
                {
                    studentProgressAndRiskLevel = _dStudentProgressAndRiskLevel.GetAllStudentProgressAndRiskLevel(new[] { currentStudent.StudentProgress.Value }).FirstOrDefault();
                    if (programActual == null)
                    {
                        log.Error($"programActual not found ...");
                        return new HttpResponseMessage { Content = new StringContent($"programActual not found ...") };
                    }
                    studentProgressAndRiskLevel.InactiveCounter = Math.Max(studentProgressAndRiskLevel.InactivityCounterConsecutive.HasValue ? (sbyte)studentProgressAndRiskLevel.InactivityCounterConsecutive.Value : 0,
                        studentProgressAndRiskLevel.InactivityCounterAcademicYear.HasValue ? (sbyte)studentProgressAndRiskLevel.InactivityCounterAcademicYear.Value : 0);
                }
                List<StudentActionOutput> DecisionRecord = GetDecisionRecord(currentStudent, decisionType,_dCache,source, xmlGuidsEnumerations,_bCache, CaseInfoList, currentAndFutureTerms, programActual, studentProgressAndRiskLevel, programRequested, log);

                #endregion

                if (matchingdecisionRecords == true)
                {
                        return req.CreateResponse(HttpStatusCode.OK, DecisionRecord);
                }
                return req.CreateResponse(HttpStatusCode.OK, new List<StudentActionOutput>() { DecisionRecord.FirstOrDefault() });
            }
            catch (Exception ex)
            {
                log.Error($"An error occurred while executing {nameof(StudentAction)}. Error details: {ex.Message}");
                return new HttpResponseMessage { StatusCode = HttpStatusCode.BadRequest, Content = new StringContent($"An error occurred while executing {nameof(StudentAction)}.. Error details: {ex.Message}") };
            }
        }

        /// <summary>
        /// Gets decisions record outputs
        /// </summary>
        /// <param name="GetDecisionRecord">list of all decisions record</param>
        /// <returns></returns>
        private static List<StudentActionOutput> GetDecisionRecord(StudentInfo currentStudent, List<DecisionData> decisionRecord, DCache dCache, PortalEnums.DecisionTableSource source , List<XMLGuidsEnumerationData> xmlGuidsEnumerations, BCache _bCache, List<CaseInfo> CaseInfoList, List<TermData> currentAndFutureTerms,ProgramData programActual,StudentProgressAndRiskLevelData studentProgressAndRiskLevel,ProgramData programRequested, TraceWriter log)
        {
            try
            {
                // get current and future terms


                //get current student current,previous,next registrations  consuming azure function
                var studentPrevCurrentNextRegs = RetrieveCurrentStudentPrevCurrentNextRegistrations(currentStudent, log) ?? new List<Model.PrevCurrentNextTermRegistrationDocument>();
                var studentCurrentTermTrans = RetrieveStudentTransactionsByTerm(currentStudent, _nextTerm.TermID.Value, log) ?? new List<PrevCurrentNextTermTransactionDocument>();
                Guid? _registrationOpenStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationstatus" && g.LookupInstance == "Open")?.LookupGuid;
                Guid? _registrationCloseStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationstatus" && g.LookupInstance == "Closed")?.LookupGuid;
                Guid? _registrationRegisteredSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationsubstatus" && g.LookupInstance == "Registered")?.LookupGuid;
                Guid? _registrationfinalgradeWithdrawalId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationfinalgrade" && g.LookupInstance == "Withdrawal (W)")?.LookupGuid;
                Guid? _registrationfinalgradeDroppedlId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationfinalgrade" && g.LookupInstance == "Dropped (DR)")?.LookupGuid;
                Guid? _registrationgradedSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationsubstatus" && g.LookupInstance == "Graded")?.LookupGuid;
                _arDivision = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_division" && g.LookupInstance == "UoPeopleAR")?.LookupGuid;
                 _enDivision = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_division" && g.LookupInstance == "UoPeople")?.LookupGuid;

                #region check if current student birthday is today
                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.StudentId} , {nameof(currentStudent.DateOfBirth)} = {currentStudent.DateOfBirth}");
                var decisionIsBirthday = CheckIfTodayIsBirthday(currentStudent.DateOfBirth, currentStudent, log) ? PortalEnums.DecisionTableIsBirthday.Yes : PortalEnums.DecisionTableIsBirthday.No;
                #endregion 

                #region check if Is first day of term
                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.StudentId} , {nameof(currentStudent.DateOfBirth)} = {currentStudent.DateOfBirth}");
               var decisionIsFirstDayOfTerm = CheckIfIsFirstDayOfTerm(_currentTerm.startDate, _currentTerm, log) ? PortalEnums.IsFirstDayOfTerm.Yes : PortalEnums.IsFirstDayOfTerm.No;
                #endregion

                #region check current student term starting 
                // check current student term starting 
                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , term starting id = {"currentStudent.StartingTerm.TermID"}");
                var decisionTermStarting = (PortalEnums.DecisionTableTermStarting?)null;

                if (currentStudent.TermStarting != null && currentStudent.TermStarting != Guid.Empty)
                {
                    if (_currentTerm.TermID == currentStudent.TermStarting)
                        decisionTermStarting = PortalEnums.DecisionTableTermStarting.Current;

                    else if (currentAndFutureTerms.FirstOrDefault(t => t.TermID == currentStudent.TermStarting) != null)
                        decisionTermStarting = PortalEnums.DecisionTableTermStarting.Upcoming;

                    else
                        decisionTermStarting = PortalEnums.DecisionTableTermStarting.Passed;
                }
                #endregion

                #region get CGPA Value for current student
                // String _CGPAChangeStatus = GetCGPAChangeStatus(currentStudent, log);
                // var CGPAChangeStatus = _CGPAChangeStatus == "100000000" ? PortalEnums.CGPAchangeStatus.NoSignificantChange : _CGPAChangeStatus == "100000001" ? PortalEnums.CGPAchangeStatus.Increased : PortalEnums.CGPAchangeStatus.Decreased;
                _CGPA= GetCGPAValue(currentStudent, log);
                #endregion

                #region check if current student has open registrations or current term
                //check if current student has open registrations or current term
                if (_registrationOpenStatusId == null || _registrationOpenStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve registration 'Open' status Guid");
                //"Registered" registration sub status

                if (_registrationRegisteredSubStatusId == null || _registrationRegisteredSubStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve registration 'Registered' sub status Guid");
                var studentRegsForCurrentTerm = (from reg in studentPrevCurrentNextRegs
                                                 where reg.Term == _currentTerm.TermID
                                                 && reg.RegistrationStatus == _registrationOpenStatusId
                                                 && reg.RegistrationSubStatus == _registrationRegisteredSubStatusId
                                                 select reg).ToList();
                ////_log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {studentRegsForCurrentTerm.Count} registrations for current term");
                var decisionHasOpenRegsCurrentTerm = studentRegsForCurrentTerm.Count > 0 ? PortalEnums.DecisionHasOpenRegsCurrentTerm.Yes : PortalEnums.DecisionHasOpenRegsCurrentTerm.No;

                #endregion

                #region check if current student Has due now transactions for registrations
                var currentOpenTransForRegistrations = (from trans in studentCurrentTermTrans

                                                        join reg in studentPrevCurrentNextRegs
                                                        on trans.RegistrationId equals reg.CrmRegistrationId

                                                        where trans.Term == _currentTerm.TermID &&
                                                              trans.TransactionStatus == (int)PortalEnums.TransactionStatus.Open &&
                                                              trans.TransactionSubStatus == (int)PortalEnums.TransactionSubStatus.DueNow &&
                                                              trans.RegistrationId != null &&
                                                              trans.RegistrationId != Guid.Empty &&
                                                              reg.RegistrationStatus == _registrationOpenStatusId &&
                                                              reg.RegistrationSubStatus == _registrationRegisteredSubStatusId &&
                                                              reg.RegistrationCrmStatus == (int)PortalEnums.RegistraruionCRMStatus.Active

                                                        select trans).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {currentOpenTransForRegistrations.Count} due now transactions for registrations");

                var decisionHasDueNowTransForRegs = currentOpenTransForRegistrations.Count > 0 ?
                    PortalEnums.DecisionTable_HasDueNowTransactionsForRegistrations.Yes :
                    PortalEnums.DecisionTable_HasDueNowTransactionsForRegistrations.No;

                #endregion  

                #region check if current student Has Oer due transactions for registrations
                var currentOpenTransForRegistrationsOverDue = (from trans in studentCurrentTermTrans

                                                        join reg in studentPrevCurrentNextRegs
                                                        on trans.RegistrationId equals reg.CrmRegistrationId

                                                        where trans.Term == _currentTerm.TermID &&
                                                              trans.TransactionStatus == (int)PortalEnums.TransactionStatus.Open &&
                                                              trans.TransactionSubStatus == (int)PortalEnums.TransactionSubStatus.Overdue &&
                                                              trans.RegistrationId != null &&
                                                              trans.RegistrationId != Guid.Empty &&
                                                              reg.RegistrationStatus == _registrationOpenStatusId &&
                                                              reg.RegistrationSubStatus == _registrationRegisteredSubStatusId &&
                                                              reg.RegistrationCrmStatus == (int)PortalEnums.RegistraruionCRMStatus.Active

                                                        select trans).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {currentOpenTransForRegistrations.Count} due now transactions for registrations");

                var decisionHasOverDueTransForRegs = currentOpenTransForRegistrationsOverDue.Count > 0 ?
                    PortalEnums.HasOverDueTransaction.Yes :
                    PortalEnums.HasOverDueTransaction.No;

                #endregion

                #region check if current student has pending registrations for next term
                //check if current student has pending registrations for next term

                Guid? registrationRequestPendingSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationsubstatus" && g.LookupInstance == "Request pending")?.LookupGuid;

                if (registrationRequestPendingSubStatusId == null || registrationRequestPendingSubStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve registration 'Request pending' sub status Guid");

                var studentPendingRegsForNextTerm = (from reg in studentPrevCurrentNextRegs
                                                     where reg.Term == _nextTerm.TermID
                                                     && reg.RegistrationStatus == _registrationOpenStatusId
                                                     && reg.RegistrationSubStatus == registrationRequestPendingSubStatusId
                                                     select reg).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {studentPendingRegsForNextTerm.Count} pending registrations for next term");

                var decisionHasPendingRegsNextTerm = studentPendingRegsForNextTerm.Count > 0 ? PortalEnums.DecisionHasPendingRegsNextTerm.Yes : PortalEnums.DecisionHasPendingRegsNextTerm.No;

                #endregion

                #region check if current student has pending registrations and no proctor assigned for next term
                //check if current student has pending registrations and no proctor assigned for next term
                Guid? CourseTypeRequiredProctoredId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_coursetype" && g.LookupInstance == "Required- Proctored")?.LookupGuid;

                if (CourseTypeRequiredProctoredId == null || CourseTypeRequiredProctoredId == Guid.Empty)
                    throw new Exception("failed to retrieve course type 'Required - proctored' status Guid");

                //"No proctor assigned yet" proctor sub status
                Guid? proctorNotAssignedYetSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_proctorsubstatus" && g.LookupInstance == "No proctor assigned yet")?.LookupGuid;

                if (proctorNotAssignedYetSubStatusId == null || proctorNotAssignedYetSubStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve proctor 'No proctor assigned yet' sub status Guid");


                var studentNoAssignedProctorPendingRegsForNextTerm = (from reg in studentPrevCurrentNextRegs
                                                                      where reg.Term == _nextTerm.TermID
                                                                      && reg.RegistrationStatus == _registrationOpenStatusId
                                                                      && reg.RegistrationSubStatus == registrationRequestPendingSubStatusId
                                                                      && reg.CourseType == CourseTypeRequiredProctoredId
                                                                      && (reg.ProctorStatus == proctorNotAssignedYetSubStatusId || reg.ProctorStatus == null)
                                                                      select reg).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {studentNoAssignedProctorPendingRegsForNextTerm.Count} pending registrations with not assigned proctors for next term");

                var decisionHasPendingRegsAndNoAssignedProctorNextTerm = studentNoAssignedProctorPendingRegsForNextTerm.Count > 0 ? PortalEnums.DecisionHasPendingRegsAndNoProctorAssignedNextTerm.Yes : PortalEnums.DecisionHasPendingRegsAndNoProctorAssignedNextTerm.No;

                #endregion

                #region check if current student has processing registrations for next term
                //check if current student has processing registrations for next term

                //"Processing Request" registration sub status
                Guid? registrationProcessingRequestSubStatusId = xmlGuidsEnumerations.FirstOrDefault(g => g.LookupEntity == "new_registrationsubstatus" && g.LookupInstance == "Processing Request")?.LookupGuid;

                if (registrationProcessingRequestSubStatusId == null || registrationProcessingRequestSubStatusId == Guid.Empty)
                    throw new Exception("failed to retrieve registration 'Processing Request' sub status Guid");

                var studentProcessingRegistrationsForNextTerm = (from reg in studentPrevCurrentNextRegs
                                                                 where reg.Term == _nextTerm.TermID
                                                                 && reg.RegistrationStatus == _registrationOpenStatusId
                                                                 && (reg.RegistrationSubStatus == registrationProcessingRequestSubStatusId || reg.RegistrationSubStatus == _registrationRegisteredSubStatusId)
                                                                 select reg).ToList();

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {studentProcessingRegistrationsForNextTerm.Count} processing registrations for next term");

                var decisionHasProcessingRegistrationsNextTerm = studentProcessingRegistrationsForNextTerm.Count > 0 ? PortalEnums.DecisionHasProcessingRegsNextTerm.Yes : PortalEnums.DecisionHasProcessingRegsNextTerm.No;
                #endregion

                #region check if current student has Number of remaining inactive terms – consecutive
                BTerm bTerm = new BTerm(log);
                string TA_CONSECUTIVE_RULE = "CounterOfInactiveConsecutiveTerms";
                string TA_ACADEMIC_YEAR_RULE = "CounterOfInactiveTermsInAcademicYear";
                var consecutiveRule = _bCache.GetRule(TA_CONSECUTIVE_RULE);
                var academicYearRule = _bCache.GetRule(TA_ACADEMIC_YEAR_RULE);

                var termActivity = bTerm.GetTermActivityByStudentId(currentStudent.CRMID)?
                    .Where(ta => ta.term?.TermID == _currentTerm.TermID)
                    .OrderByDescending(x => x.term.termNumber)
                    .FirstOrDefault();

                var ruleValueOfConsecutive = Int32.Parse(consecutiveRule.RuleValue?.ToString());
                var ruleValueOfAcademicYear = Int32.Parse(academicYearRule.RuleValue?.ToString());

                var taValueOfConsecutive = termActivity?.inactivityCounterConsecutive ?? 0;
                var taValueOfAcademicYear = termActivity?.inactivityCounterOnAcademicYear ?? 0;

                var decisionInactivityCounterConsecutive = ruleValueOfConsecutive - taValueOfConsecutive;
                var decisionInactivityCounterOnAcademicYear = ruleValueOfAcademicYear - taValueOfAcademicYear;

                log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID}, term-activity = {termActivity?.termActivityId}, " +
                    $"{nameof(ruleValueOfConsecutive)} = {ruleValueOfConsecutive}, {nameof(taValueOfConsecutive)} = {taValueOfConsecutive}, " +
                    $"{nameof(ruleValueOfAcademicYear)} = {ruleValueOfAcademicYear}, {nameof(taValueOfAcademicYear)} = {taValueOfAcademicYear}");
                #endregion

                #region check if current student has Approved LOA case – Current term
                var decisionHasApprovedLOAcaseCurrentTerm = CaseInfoList.Count > 0 ? PortalEnums.DecisionHasApprovedLOAcaseCurrentTerm.Yes : PortalEnums.DecisionHasApprovedLOAcaseCurrentTerm.No;
                #endregion

                #region check if current student Dropped or withdrawal from all courses
                //get all document related with student id and convert to object as type student info
                var prevCurrentNextTermRegistrationList = _dMongoDB.GetDocumentsFromRegistrationsPrevCurrentNextTermCollection(currentStudent.CRMID, _currentTerm.TermID).ToList<PrevCurrentNextTermRegistration>();
                bool status = false;
                if (prevCurrentNextTermRegistrationList.Count > 0)
                {
                    var prevCurrentNextTermRegistrationOpenStatus = (from reg in prevCurrentNextTermRegistrationList
                                                                     where reg.Term == _currentTerm.TermID
                                                                     && reg.RegistrationStatus == _registrationOpenStatusId
                                                                     select reg).ToList();

                    if (prevCurrentNextTermRegistrationOpenStatus.Count != 0)
                    {
                        status = false;
                    }
                    else
                    {
                        var prevCurrentNextTermRegistrationCloseStatus = (from reg in prevCurrentNextTermRegistrationList
                                                                          where reg.Term == _currentTerm.TermID
                                                                            && reg.RegistrationStatus == _registrationCloseStatusId
                                                                            && reg.RegistrationSubStatus == _registrationgradedSubStatusId
                                                                            && ((reg.FinalGrade == _registrationfinalgradeWithdrawalId) || (reg.FinalGrade == _registrationfinalgradeDroppedlId))
                                                                          select reg).ToList();
                        if (prevCurrentNextTermRegistrationCloseStatus.Count > 0)
                        {
                            status = true;
                        }
                        else
                        {
                            status = false;
                        }
                    }
                }
                else
                {

                }
                var decisionHasDroppedOrWithdrawalFromAllCourses = status == true ? PortalEnums.DecisionHasDroppedorWithDrawalFromallCourses.Yes : PortalEnums.DecisionHasDroppedorWithDrawalFromallCourses.No;
                #endregion


                #region Prepare Registration Period
                var currnetDate = DateTime.UtcNow;
                PortalEnums.RegistrationPeriod  RegistrationPeriod;
                if (currnetDate > currentStudent.DateToOpenEarlyRegistrationForNextTerm  && currnetDate < currentStudent.DateToCloseEarlyRegistrationForNextTerm )
                {
                    RegistrationPeriod = PortalEnums.RegistrationPeriod.DuringEarly;
                }
                else if (currnetDate > currentStudent.DateToOpenLateRegistrationForNextTerm && currnetDate < currentStudent.DateToCloseLateRegistrationForNextTerm)
                {
                    RegistrationPeriod = PortalEnums.RegistrationPeriod.DuringLate;
                }
                else if (currnetDate > currentStudent.DateToCloseEarlyRegistrationForNextTerm && currnetDate < currentStudent.DateToOpenLateRegistrationForNextTerm)
                {
                    RegistrationPeriod = PortalEnums.RegistrationPeriod.DuringEarlyandLate;
                }
                else
                {
                    RegistrationPeriod = PortalEnums.RegistrationPeriod._skip;
                }

                #endregion

                #region Prepare CourseCodes List 
                var CourseCodes = (from reg in studentPrevCurrentNextRegs                                               
                                         select reg.CourseCode).Distinct().ToList();


                #endregion
                //get current academic week
                AcademicWeekData currentAcademicWeek = _bCache.GetCurrentWeek(_currentTerm.TermID.Value);
                if (currentAcademicWeek == null)
                {
                    log.Error($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , failed to retrieve current academic week");
                    throw new Exception($"failed to retrieve current academic week");
                    
                }
                

               List<DecisionData> matchingPersonalMessageDecisions = (from dt in decisionRecord.Where(t => t.DecisionType.Equals((int)PortalEnums.DecisionType.SoecificMessages)).ToList()
                                                       where (dt.Division == null || dt.Division == currentStudent.Division)
                                                       && (dt.DuplicateAppInputStudentStatus == PortalEnums.StudentStatus._Skip || dt.DuplicateAppInputStudentStatus == currentStudent.StudentStatus)
                                                       && (dt.InputDegreeTypeFrom == PortalEnums.DegreeTypeDecisionTable.Skip || dt.InputDegreeTypeFrom == programRequested.InputDegreeTypeFrom)
                                                       && (dt.PaymentStyle == PortalEnums.PaymentStyle._skip || dt.PaymentStyle == currentStudent.PaymentStyle)
                                                     && (dt.OrientationCourseCreationStatus == PortalEnums.OrientationCourseCreationStatus._skip || dt.OrientationCourseCreationStatus == currentStudent.OrientationCourseCreationStatus)
                                                       && (dt.CountryofResidence.ToString().Contains(currentStudent.CountryofResidencePickList.ToString())
                                                        ||   dt.CountryofResidence == null   || dt.CountryofResidence.ToString().Contains(PortalEnums.CountryofResidencePickList._skip.ToString())     )
                                                      
                                                       && (dt.FinancialStatus == PortalEnums.StudentFinancialStatus._Skip || dt.FinancialStatus == currentStudent.FinancialStatus)
                                                       && (dt.CourseCode == null || CourseCodes.Contains(dt.CourseCode))
                                                       && (dt.RegistrationPeriod == PortalEnums.RegistrationPeriod._skip || dt.RegistrationPeriod == RegistrationPeriod)
                                                       && (dt.HasDueNowTransactionsForRegistrations == PortalEnums.DecisionTable_HasDueNowTransactionsForRegistrations._Skip || dt.HasDueNowTransactionsForRegistrations == decisionHasDueNowTransForRegs)
                                                       && (dt.HasOverDueTransaction == PortalEnums.HasOverDueTransaction._skip|| dt.HasOverDueTransaction == decisionHasOverDueTransForRegs)
                                                       && (dt.HasOpenRegsCurrentTerm == PortalEnums.DecisionHasOpenRegsCurrentTerm._Skip || dt.HasOpenRegsCurrentTerm == decisionHasOpenRegsCurrentTerm)
                                                       && (dt.HasPendingRegsNextTerm == PortalEnums.DecisionHasPendingRegsNextTerm._Skip || dt.HasPendingRegsNextTerm == decisionHasPendingRegsNextTerm)
                                                       && (dt.HasPendingRegsAndNoProctorAssignedNextTerm == PortalEnums.DecisionHasPendingRegsAndNoProctorAssignedNextTerm._Skip || dt.HasPendingRegsAndNoProctorAssignedNextTerm == decisionHasPendingRegsAndNoAssignedProctorNextTerm)
                                                       && (dt.HasProcessingRegsNextTerm == PortalEnums.DecisionHasProcessingRegsNextTerm._Skip || dt.HasProcessingRegsNextTerm == decisionHasProcessingRegistrationsNextTerm)
                                                       && (dt.HasApprovedLOAcaseCurrentTerm == PortalEnums.DecisionHasApprovedLOAcaseCurrentTerm._Skip || dt.HasApprovedLOAcaseCurrentTerm == decisionHasApprovedLOAcaseCurrentTerm)
                                                       && (dt.HasDroppedOrWithdrawalFromAllCourses == PortalEnums.DecisionHasDroppedorWithDrawalFromallCourses._Skip || dt.HasDroppedOrWithdrawalFromAllCourses == decisionHasDroppedOrWithdrawalFromAllCourses)
                                                       && (dt.IsBirthday == PortalEnums.DecisionTableIsBirthday._Skip || dt.IsBirthday == decisionIsBirthday)
                                                       && (dt.IsFirstDayOfTerm == PortalEnums.IsFirstDayOfTerm._Skip || dt.IsFirstDayOfTerm == decisionIsFirstDayOfTerm)
                                                       && (dt.InputTermStarting == PortalEnums.DecisionTableTermStarting._Skip || dt.InputTermStarting == decisionTermStarting)
                                                       && (dt.FoundationProgramGroup == PortalEnums.foundationProgramGroup._skip || dt.FoundationProgramGroup == programActual.FoundationProgramGroupFix)
                                                       && (dt.foundationType == PortalEnums.FoundationType.Skip || dt.foundationType == programActual.FoundationType)
                                                       && (dt.DegreeProgramGroupProgramRequested == PortalEnums.DegreeProgramGroupProgramRequested._skip || dt.DegreeProgramGroupProgramRequested == programRequested.DegreeProgramGroup)
                                                       && (dt.InputDegreeLevelFrom == PortalEnums.degreeLevelDecisionTable.Skip || dt.InputDegreeLevelFrom == programRequested.degreeLevel)
                                                       && (dt.programGroup == PortalEnums.ProgramGroup.Skip || dt.programGroup == programRequested.programGroup)
                                                       && (dt.InputMajorTo == PortalEnums.MajorDecisionTable.Skip || dt.InputMajorTo == programRequested.major)
                                                       && (dt.programType == PortalEnums.ProgramType._skip || dt.programType == programRequested.programType)
                                                       && (dt.DegreeProgramGroup == PortalEnums.DegreeProgramGroupProgramRequested._skip || dt.DegreeProgramGroup == programRequested.DegreeProgramGroup)
                                                       && (dt.NumberOfRemainingInactiveTerms_Consecutive == null || dt.NumberOfRemainingInactiveTerms_Consecutive == decisionInactivityCounterConsecutive)
                                                       && (dt.NumberOfRemainingInactiveTerms_AcademicYear == decisionInactivityCounterOnAcademicYear || dt.NumberOfRemainingInactiveTerms_AcademicYear == null)
                                                       && (dt.TermNumberInAcademicYear == _currentTerm.TermNumberInAcademicYear || dt.TermNumberInAcademicYear == null)
                                                       && (dt.InactiveCounter == studentProgressAndRiskLevel.InactiveCounter || dt.InactiveCounter == null)
                                                       && (dt.OnHoldCounter == studentProgressAndRiskLevel.OnHoldCounter || dt.OnHoldCounter == null)
                                                       && (dt.RiskLevel == PortalEnums.RiskLevel._skip || dt.RiskLevel == studentProgressAndRiskLevel.RiskLevel)
                                                       && (dt.RiskType == PortalEnums.RiskType._skip || dt.RiskType == studentProgressAndRiskLevel.RiskType)
                                                       && (dt.WeekFrom <= currentAcademicWeek.WeekNumber || dt.WeekFrom == null)
                                                       && (dt.WeekTo >= currentAcademicWeek.WeekNumber || dt.WeekTo == null)
                                                       && ((dt.StartDateToShow >= DateTime.UtcNow && dt.StartDateToShow <= DateTime.UtcNow.AddDays((double)dt.NumberofDaystoShow)) || dt.StartDateToShow == null)
                                                       && (_currentTerm.ShowDeadLineToPayForCourses.Value.AddDays(-(double)dt.NumberofDaystoshowbeforepaymentdeadline) >= DateTime.UtcNow  || dt.NumberofDaystoshowbeforepaymentdeadline == null)
                                                       // && (dt.CGPAchangeStatus == CGPAChangeStatus || dt.CGPAchangeStatus == null )
                                                       && (dt.Source == PortalEnums.DecisionTableSource._skip || dt.Source == source)
                                                       select dt).OrderBy(d => d.DecisionPriority).ToList();
                if (matchingPersonalMessageDecisions == null || matchingPersonalMessageDecisions.Count ==0)
                {
                    matchingPersonalMessageDecisions = (from dt in decisionRecord.Where(t => t.DecisionType.Equals((int)PortalEnums.DecisionType.StudentRetention)).ToList()
                                                        where (dt.Division == null || dt.Division == currentStudent.Division)
                                                           && (dt.DuplicateAppInputStudentStatus == PortalEnums.StudentStatus._Skip || dt.DuplicateAppInputStudentStatus == currentStudent.StudentStatus)
                                                           && (dt.FinancialStatus == PortalEnums.StudentFinancialStatus._Skip || dt.FinancialStatus == currentStudent.FinancialStatus)
                                                           && (dt.HasDueNowTransactionsForRegistrations == PortalEnums.DecisionTable_HasDueNowTransactionsForRegistrations._Skip || dt.HasDueNowTransactionsForRegistrations == decisionHasDueNowTransForRegs)
                                                           && (dt.HasOpenRegsCurrentTerm == PortalEnums.DecisionHasOpenRegsCurrentTerm._Skip || dt.HasOpenRegsCurrentTerm == decisionHasOpenRegsCurrentTerm)
                                                           && (dt.HasPendingRegsNextTerm == PortalEnums.DecisionHasPendingRegsNextTerm._Skip || dt.HasPendingRegsNextTerm == decisionHasPendingRegsNextTerm)
                                                           && (dt.HasPendingRegsAndNoProctorAssignedNextTerm == PortalEnums.DecisionHasPendingRegsAndNoProctorAssignedNextTerm._Skip || dt.HasPendingRegsAndNoProctorAssignedNextTerm == decisionHasPendingRegsAndNoAssignedProctorNextTerm)
                                                           && (dt.HasProcessingRegsNextTerm == PortalEnums.DecisionHasProcessingRegsNextTerm._Skip || dt.HasProcessingRegsNextTerm == decisionHasProcessingRegistrationsNextTerm)
                                                           && (dt.HasApprovedLOAcaseCurrentTerm == PortalEnums.DecisionHasApprovedLOAcaseCurrentTerm._Skip || dt.HasApprovedLOAcaseCurrentTerm == decisionHasApprovedLOAcaseCurrentTerm)
                                                           && (dt.HasDroppedOrWithdrawalFromAllCourses == PortalEnums.DecisionHasDroppedorWithDrawalFromallCourses._Skip || dt.HasDroppedOrWithdrawalFromAllCourses == decisionHasDroppedOrWithdrawalFromAllCourses)
                                                           && (dt.IsBirthday == PortalEnums.DecisionTableIsBirthday._Skip || dt.IsBirthday == decisionIsBirthday)
                                                           && (dt.InputTermStarting == PortalEnums.DecisionTableTermStarting._Skip || dt.InputTermStarting == decisionTermStarting)                                                          
                                                           && (dt.NumberOfRemainingInactiveTerms_Consecutive == null || dt.NumberOfRemainingInactiveTerms_Consecutive == decisionInactivityCounterConsecutive)
                                                           && (dt.NumberOfRemainingInactiveTerms_AcademicYear == decisionInactivityCounterOnAcademicYear || dt.NumberOfRemainingInactiveTerms_AcademicYear == null)
                                                           && (dt.TermNumberInAcademicYear == _currentTerm.TermNumberInAcademicYear || dt.TermNumberInAcademicYear == null)
                                                           && (dt.WeekFrom <= currentAcademicWeek.WeekNumber || dt.WeekFrom == null)
                                                           && (dt.WeekTo >= currentAcademicWeek.WeekNumber || dt.WeekTo == null)
                                                           // && (dt.CGPAchangeStatus == CGPAChangeStatus || dt.CGPAchangeStatus == null )
                                                           && (dt.Source == PortalEnums.DecisionTableSource._skip || dt.Source == source)
                                                           select dt).OrderBy(d => d.DecisionPriority).ToList();
                }
                //Get relevant top banner decisions for current student ordered by priority
             
                    log.Info($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , found {matchingPersonalMessageDecisions.Count} top banner decisions for this student");
                    return matchingPersonalMessageDecisions.Select(d => new StudentActionOutput
                    {
                        ActionTitle = (d.ActionTitle) ?? "",
                        ActionTitleAR = (d.ActionTitle_ar) ?? "",
                        ActionLink = d.ActionLink ?? "",
                        MessageText = (d.OutputMessageToShow != null) ? DynamicFieldsHandler(d.OutputMessageToShow,currentStudent) : "",
                        MessageTextAR = (d.OutputMessageToShow_AR != null) ? DynamicFieldsHandler(d.OutputMessageToShow_AR,currentStudent) : "",
                        MessageTitle = (d.MessageTitle != null) ? DynamicFieldsHandler(d.MessageTitle,currentStudent) : "",
                        MessageTitleAR = (d.MessageTitle_ar != null) ? DynamicFieldsHandler(d.MessageTitle_ar,currentStudent) : "",
                        Slug = (d.Slug) ?? "",
                        BackgroundImage = (d.BackgroundImage) ?? ""
                    }).ToList();
                
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , failed to retrieve Decision record");
                throw new Exception($"failed to retrieve records");
            }
        }

        // check if current student berthday is today 
        private static bool CheckIfTodayIsBirthday(DateTime? birthDate, StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                var currentDay = DateTime.Now.Day;
                var currentMonth = DateTime.Now.Month;


                return birthDate?.Day == currentDay && birthDate?.Month == currentMonth;
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(CheckIfTodayIsBirthday)}] - an error occured ,  student id = {currentStudent.CRMID} , ex : {ex}");
                throw;
            }
        }
        private static bool CheckIfIsFirstDayOfTerm(DateTime? startDate, TermData currentTerm, TraceWriter log)
        {
            try
            {
                var currentDay = DateTime.Now.Day;
                var currentMonth = DateTime.Now.Month;


                return startDate?.Day == currentDay && startDate?.Month == currentMonth;
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(CheckIfTodayIsBirthday)}] - an error occured ,  student id = {currentTerm.TermID} , ex : {ex}");
                throw;
            }
        }

        /// <summary>
        /// This method retrieves previous , current , next registrations for current student by consuming Azure function, in which it reads 
        /// the data from Mongo DB
        /// </summary>
        /// <returns></returns>
        private static List<Model.PrevCurrentNextTermRegistrationDocument> RetrieveCurrentStudentPrevCurrentNextRegistrations(StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                WebAPIHelper _apiHelper = new WebAPIHelper(log);
                var uriBaseAddress = Environment.GetEnvironmentVariable("registrationUrl");
                var uriPath = "api/RetrievePreviousCurrentNextTermRegistrations";
                var TOKEN_NAME = Environment.GetEnvironmentVariable("registrationUrlHeaderTokenName");
                var TOKEN = Environment.GetEnvironmentVariable("registrationUrlHeaderToken");
                var parameters = new Dictionary<string, string>
                {
                    { "StudentId" , currentStudent.CRMID.ToString()}
                };

                var headers = new Dictionary<string, string>();
                headers.Add(TOKEN_NAME, TOKEN);

                var studentPrevCurrentNextRegistrations = _apiHelper.Get<List<Model.PrevCurrentNextTermRegistrationDocument>>(uriBaseAddress, uriPath, parameters, null, headers);
                log.Info($"[{nameof(RetrieveCurrentStudentPrevCurrentNextRegistrations)}] - number of regs in {studentPrevCurrentNextRegistrations}  = {studentPrevCurrentNextRegistrations.Count} , student id = {currentStudent.StudentId}");
                return studentPrevCurrentNextRegistrations;
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(RetrieveCurrentStudentPrevCurrentNextRegistrations)}] - an error occured , student id = {currentStudent.StudentId} , ex : {ex}");
                throw;
            }
        }

        /// <summary>
        /// This method retrieves term transactions for current student by consuming Azure function, in which it reads 
        /// the data from Mongo DB
        /// </summary>
        /// <returns></returns>
        private static List<PrevCurrentNextTermTransactionDocument> RetrieveStudentTransactionsByTerm(StudentInfo currentStudent, Guid termId, TraceWriter log)
        {
            try
            {
                WebAPIHelper _apiHelper = new WebAPIHelper(log);
                var uriBaseAddress = Environment.GetEnvironmentVariable("transactionUrl");
                var uriPath = "api/RetrievePreviousCurrentNextTermTransactions";
                var TOKEN_NAME = Environment.GetEnvironmentVariable("transactionUrlHeaderTokenName");
                var TOKEN = Environment.GetEnvironmentVariable("transactionUrlHeaderToken");
                var parameters = new Dictionary<string, string>
                {
                    { "StudentId" , currentStudent.CRMID.ToString() }
                };
                    var headers = new Dictionary<string, string>
                {
                    { TOKEN_NAME ,TOKEN}
                };

                var studentPrevCurrentNextTransactions = _apiHelper.Get<List<PrevCurrentNextTermTransactionDocument>>(uriBaseAddress, uriPath, parameters, null, headers);
                log.Info($"[{nameof(RetrieveStudentTransactionsByTerm)}] - number of trans in {nameof(studentPrevCurrentNextTransactions)}  = {studentPrevCurrentNextTransactions.Count} , student id = {currentStudent.StudentId}, term Id = {termId}");
                return studentPrevCurrentNextTransactions;
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(RetrieveStudentTransactionsByTerm)}] - an error occured when get Transactions By Term for , student id = {currentStudent.StudentId}, term id = {termId}, ex : {ex}");
                throw;
            }
            
        }

        /// <summary>
        /// This method retrieves current and future terms 
        /// the data from cash
        /// </summary>
        /// <returns></returns>
        private static List<TermData> GetCurrentAndFutureTerms(StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                // get current and future term
                BCache _bCache = new BCache(log);
                List<TermData> _currentAndFutureTerms = _bCache.GetCurrentAndFutureTerms();
                if (_currentAndFutureTerms == null || _currentAndFutureTerms.Count < 2)
                {
                    log.Error($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , failed to retrieve current & future terms");
                    throw new Exception($"failed to retrieve current term");
                }
                else
                {
                    return _currentAndFutureTerms;
                }

            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(GetDecisionRecord)}] - student id = {currentStudent.CRMID} , failed to retrieve current & future terms");
                throw new Exception($"failed to retrieve current term");

            }
        }

        /// <summary>
        /// This method retrieves CGBA 
        /// the data from new term statistics
        /// </summary>
        /// <returns></returns>
        /// 
        private static String GetCGPAChangeStatus(StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                UOPeopleCRMConnection _crmConnection = new UOPeopleCRMConnection(log);
                XrmServiceContext _service = _crmConnection.getserviceProvider();
                using (var provider = _crmConnection.getserviceProvider())
                {
                    var term = (from a in provider.New_timeperiodSet
                                join c in provider.New_termstatisticsSet
                                on a.New_timeperiodId.Value equals c.new_termid.Id
                                where (c.new_studentid.Id == new Guid("134d10f3-4830-e911-a976-000d3a1998fb"))
                                orderby a.New_TermNumber descending
                                select new
                                {
                                    New_TermNumber = a.New_TermNumber,
                                    CGPAchangeStatus =c.new_CgpaChangeStatus
                                }
                               ).ToList().FirstOrDefault();
                    return term.CGPAchangeStatus.Value.ToString();
                }
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(GetCGPAChangeStatus)}] - student id = {currentStudent.CRMID} , failed to retrieve CGPA change status current terms");
                throw new Exception($"failed to retrieve CGPA change status current term");
            }
        }

        private static double? GetCGPAValue(StudentInfo currentStudent, TraceWriter log)
        {
            try
            {
                UOPeopleCRMConnection _crmConnection = new UOPeopleCRMConnection(log);
                XrmServiceContext _service = _crmConnection.getserviceProvider();
                using (var provider = _crmConnection.getserviceProvider())
                {
                    var term = (from a in provider.New_timeperiodSet
                                join c in provider.New_termstatisticsSet
                                on a.New_timeperiodId.Value equals c.new_termid.Id
                                where (c.new_studentid.Id == currentStudent.CRMID.Value)
                                orderby a.New_TermNumber descending
                                select new
                                {
                                    New_TermNumber = a.New_TermNumber,
                                    CGPAchangeStatus = c.new_CgpaChangeStatus,
                                    CGPA=c.New_CGPA
                                }
                               ).ToList().FirstOrDefault();
                    if( term != null )
                    {

                        if (term.CGPA.HasValue)
                        {
                            return  (double)term.CGPA.Value;
                        }
                        else
                        {
                            return  null;
                        }
                    }
                    else
                    {
                        return null;
                    }
                }
            }
            catch (Exception ex)
            {
                log.Error($"[{nameof(GetCGPAChangeStatus)}] - student id = {currentStudent.CRMID} , failed to retrieve CGPA change status current terms");
                throw new Exception($"failed to retrieve CGPA change status current term");
            }
        }
        private static string DynamicFieldsHandler (string inputString, StudentInfo currentStudent)
        {

            string studentNamePlaceholder = "Student_Name";
            string studentNameValue =null;
            string studentPreferredNamePlaceholder = "Student_Preferred_Name";
            string studentPreferredNameValue = null; ;
            string studentCountryPlaceholder = "Student_Country";
            string studentCountryValue = null; ;
            string programPlaceholder = "Program";
            string programValue = null; ;
            string degreePlaceholder = "Degree";
            string degreeValue = null; ;
            if (currentStudent.Division.Value != null && currentStudent.Division.Value != Guid.Empty)
            {
            if(currentStudent.Division.Value == _arDivision)
                {
                    studentNameValue = "John Doe";
                    studentPreferredNameValue = "John Doe";
                    studentCountryValue = "John Doe";
                    programValue = "John Doe";
                    degreeValue = "John Doe";
                }  
            else if(currentStudent.Division.Value == _enDivision)

                {
                    studentNameValue = "John Doe";
                    studentPreferredNameValue = "John Doe";
                    studentCountryValue = "John Doe";
                    programValue = "John Doe";
                    degreeValue = "John Doe";
                }
                }
            
            string firstDayOfTheUpcomingTermPlaceholder = "First_Day_Of_The_Upcoming_Term";
            string firstDayOfTheUpcomingTermValue = _nextTerm?.startDate.Value.ToString();
            string lastDayOfTheUpcomingTermPlaceholder = "Last_Day_Of_The_Upcoming_Term";
            string lastDayOfTheUpcomingTermValue = _nextTerm?.endDate.Value.ToString();
            string registrationPeriodDatesPlaceholder = "Registration_Period_Dates";
            string registrationPeriodDatesValue = "Early=";
            string lateRegistrationPeriodDatesPlaceholder = "Late_Registration_Period_Dates";
            string lateRegistrationPeriodDatesValue = "John Doe";
            string studentsRegistrationPeriodDatesPlaceholder = "Students_Registration_Period_Dates";
            string studentsRegistrationPeriodDatesValue = "John Doe";
            string paymentDeadlinePlaceholder = "Payment_Deadline";
            string paymentDeadlineValue = _currentTerm?.ShowDeadLineToPayForCourses.Value.ToString();
            string CGPAPlaceholder = "CGPA";
            string CGPAValue = _CGPA.ToString();
            string maxCoursesAllowedPlaceholder = "Max_Courses_Allowed";
            string maxCoursesAllowedValue = currentStudent?.CourseLoadAllowed.Value.ToString();
            string orientationDeadlinePlaceholder = "Orientation_Deadline";
            string orientationDeadlineValue = "John Doe";
            ; List<object> replacements = new List<object>()
                                      {
                                     $"{studentNamePlaceholder}:{studentNameValue}",
                                     $"{studentPreferredNamePlaceholder}:{studentPreferredNameValue}",
                                     $"{studentCountryPlaceholder}:{studentCountryValue}",
                                     $"{programPlaceholder}:{programValue}",
                                     $"{degreePlaceholder}:{degreeValue}",
                                     $"{firstDayOfTheUpcomingTermPlaceholder}:{firstDayOfTheUpcomingTermValue}",
                                     $"{lastDayOfTheUpcomingTermPlaceholder}:{lastDayOfTheUpcomingTermValue}",
                                     $"{registrationPeriodDatesPlaceholder}:{registrationPeriodDatesValue}",
                                     $"{lateRegistrationPeriodDatesPlaceholder}:{lateRegistrationPeriodDatesValue}",
                                     $"{studentsRegistrationPeriodDatesPlaceholder}:{studentsRegistrationPeriodDatesValue}",
                                     $"{paymentDeadlinePlaceholder}:{paymentDeadlineValue}",
                                     $"{CGPAPlaceholder}:{CGPAValue}",
                                     $"{maxCoursesAllowedPlaceholder}:{maxCoursesAllowedValue}",
                                     $"{orientationDeadlinePlaceholder}:{orientationDeadlineValue}",
                                  
                                    };

            return ReplacePlaceholders(inputString, replacements);
        }
        private static string ReplacePlaceholders(string inputString, List<object> replacements)
        {
            foreach (object replacement in replacements)
            {
                string placeholder = "#" + replacement.ToString().Split(':')[0] + "#";
                string value = replacement.ToString().Split(':')[1]; inputString = inputString.Replace(placeholder, value);
            }
            return inputString;
        }



    }
}
