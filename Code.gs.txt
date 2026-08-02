/******************************************************
 * SMART CLASSROOM V1 — REFACTORED BACKEND
 * Google Apps Script + Google Sheets
 *
 * Students columns:
 * A ID, B No, C Firstname, D Nickname, E Present,
 * F Absent, G Attendance %, H Exam Score, I Result, J PIN
 *
 * Attendance columns:
 * A ID, B Nickname, C onward = dates such as 01-Jul
 ******************************************************/

const CONFIG = Object.freeze({
  SHEETS: Object.freeze({
    STUDENTS: 'Students',
    ATTENDANCE: 'Attendance',
    SETTINGS: 'Settings',
    HOLIDAY: 'Holiday'
  }),
  STUDENT_COLUMNS: Object.freeze({
    ID: 1,
    NO: 2,
    FIRSTNAME: 3,
    NICKNAME: 4,
    PRESENT: 5,
    ABSENT: 6,
    ATTENDANCE: 7,
    EXAM_SCORE: 8,
    RESULT: 9,
    PIN: 10
  }),
  PASS_RULES: Object.freeze({
    ATTENDANCE_PERCENT: 70,
    EXAM_SCORE: 25
  }),
  DEFAULT_PAGE: 'Index',
  ALLOWED_PAGES: Object.freeze(['Index', 'Login', 'Teacher', 'Student', 'QR']),
  DEFAULT_TIMEZONE: 'Asia/Bangkok'
});

function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('🛠️ ระบบเช็คชื่อ')
    .addItem('📅 สร้างปฏิทินเช็คชื่อ', 'generateCalendar')
    .addItem('📊 คำนวณผลการเรียนใหม่', 'calculateResult')
    .addToUi();
}

function doGet(e) {
  try {
    const requestedPage = e && e.parameter && e.parameter.page
      ? String(e.parameter.page).trim()
      : CONFIG.DEFAULT_PAGE;

    const page = CONFIG.ALLOWED_PAGES.includes(requestedPage)
      ? requestedPage
      : CONFIG.DEFAULT_PAGE;

    return HtmlService
      .createTemplateFromFile(page)
      .evaluate()
      .setTitle('Smart Classroom V1')
      .addMetaTag('viewport', 'width=device-width, initial-scale=1')
      .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);

  } catch (error) {
    return HtmlService.createHtmlOutput(
      '<h3>เกิดข้อผิดพลาดในการโหลดแอป</h3>' +
      '<p>' + escapeHtml_(error.message) + '</p>' +
      '<p>กรุณาตรวจสอบว่ามีไฟล์ HTML ที่ระบบเรียกใช้อยู่</p>'
    );
  }
}

function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent();
}

function getSheetSafe_(name) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(name);
  if (!sheet) throw new Error("ไม่พบ Sheet ชื่อ '" + name + "'");
  return sheet;
}

function normalizeId_(value) {
  return String(value == null ? '' : value).trim();
}

function normalizePin_(value) {
  return String(value == null ? '' : value).trim();
}

function calculateStudentAttendance_(student) {
  const present = Number(student.present) || 0;
  const absent = Number(student.absent) || 0;
  const total = present + absent;

  if (total <= 0) {
    return "0%";
  }

  return Math.round((present / total) * 100) + "%";
}

function isPresent_(value) {
  const text = String(value == null ? '' : value).trim().toLowerCase();
  return value === true || value === 1 ||
    text === 'true' || text === 'มา' || text === 'present' || text === 'p';
}

function getTimeZone_() {
  return SpreadsheetApp.getActiveSpreadsheet().getSpreadsheetTimeZone() ||
    Session.getScriptTimeZone() || CONFIG.DEFAULT_TIMEZONE;
}

function parseDate_(value) {
  if (Object.prototype.toString.call(value) === '[object Date]' && !isNaN(value.getTime())) {
    return new Date(value.getTime());
  }

  const text = String(value == null ? '' : value).trim();
  if (!text) throw new Error('ไม่พบวันที่');

  const match = text.match(/^(\d{4})-(\d{2})-(\d{2})$/);
  if (match) {
    return new Date(Number(match[1]), Number(match[2]) - 1, Number(match[3]), 12, 0, 0);
  }

  const date = new Date(text);
  if (isNaN(date.getTime())) throw new Error('รูปแบบวันที่ไม่ถูกต้อง: ' + text);
  return date;
}

function formatDate_(value, format) {
  return Utilities.formatDate(parseDate_(value), getTimeZone_(), format);
}

function getDateKey_(value) {
  return formatDate_(value, 'yyyy-MM-dd');
}

function getShortDate_(value) {
  return formatDate_(value, 'dd-MMM');
}

function escapeHtml_(text) {
  return String(text == null ? '' : text)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}

function success_(message, data) {
  return { success: true, message: message || '', data: data === undefined ? null : data };
}

function failure_(message) {
  return { success: false, message: message || 'เกิดข้อผิดพลาด', data: null };
}

function findAttendanceColumn_(sheet, date) {
  const lastCol = sheet.getLastColumn();
  if (lastCol < 3) return -1;

  const headers = sheet.getRange(1, 1, 1, lastCol).getDisplayValues()[0];
  const target = getShortDate_(date).toLowerCase();

  return headers.findIndex(function(header) {
    return String(header).trim().toLowerCase() === target;
  });
}

function getInitialData() {
  try {
    const today = new Date();
    return {
      success: true,
      students: getStudents(),
      dashboard: getTodayDashboard(today),
      serverDate: formatDate_(today, 'yyyy-MM-dd')
    };
  } catch (error) {
    return { success: false, message: error.message };
  }
}

function getStudents() {
  const sheet = getSheetSafe_(CONFIG.SHEETS.STUDENTS);
  const lastRow = sheet.getLastRow();
  if (lastRow < 2) return [];

  const data = sheet.getRange(2, 1, lastRow - 1, CONFIG.STUDENT_COLUMNS.PIN).getValues();

  return data.map(function(row) {
    return {
      id: row[CONFIG.STUDENT_COLUMNS.ID - 1],
      no: row[CONFIG.STUDENT_COLUMNS.NO - 1],
      firstname: row[CONFIG.STUDENT_COLUMNS.FIRSTNAME - 1],
      nickname: row[CONFIG.STUDENT_COLUMNS.NICKNAME - 1],
      present: row[CONFIG.STUDENT_COLUMNS.PRESENT - 1],
      absent: row[CONFIG.STUDENT_COLUMNS.ABSENT - 1],
      attendance: row[CONFIG.STUDENT_COLUMNS.ATTENDANCE - 1],
      examscore: row[CONFIG.STUDENT_COLUMNS.EXAM_SCORE - 1],
      result: row[CONFIG.STUDENT_COLUMNS.RESULT - 1]
    };
  });
}

function findStudentRow_(studentId) {
  const sheet = getSheetSafe_(CONFIG.SHEETS.STUDENTS);
  const lastRow = sheet.getLastRow();
  if (lastRow < 2) return null;

  const ids = sheet.getRange(2, CONFIG.STUDENT_COLUMNS.ID, lastRow - 1, 1).getDisplayValues().flat();
  const target = normalizeId_(studentId);
  const index = ids.findIndex(function(id) { return normalizeId_(id) === target; });
  return index < 0 ? null : index + 2;
}

function getStudentById_(studentId) {
  const rowNumber = findStudentRow_(studentId);
  if (rowNumber === null) return null;

  const sheet = getSheetSafe_(CONFIG.SHEETS.STUDENTS);
  const row = sheet.getRange(rowNumber, 1, 1, CONFIG.STUDENT_COLUMNS.PIN).getValues()[0];

  return {
    id: row[CONFIG.STUDENT_COLUMNS.ID - 1],
    no: row[CONFIG.STUDENT_COLUMNS.NO - 1],
    firstname: row[CONFIG.STUDENT_COLUMNS.FIRSTNAME - 1],
    nickname: row[CONFIG.STUDENT_COLUMNS.NICKNAME - 1],
    present: row[CONFIG.STUDENT_COLUMNS.PRESENT - 1],
    absent: row[CONFIG.STUDENT_COLUMNS.ABSENT - 1],
    attendance: row[CONFIG.STUDENT_COLUMNS.ATTENDANCE - 1],
    examscore: row[CONFIG.STUDENT_COLUMNS.EXAM_SCORE - 1],
    result: row[CONFIG.STUDENT_COLUMNS.RESULT - 1],
    pin: row[CONFIG.STUDENT_COLUMNS.PIN - 1]
  };
}

function checkStudentLogin(studentId, pin) {
  try {
    const student = getStudentById_(studentId);
    if (!student) return failure_('ไม่พบรหัสนักเรียน');
    if (normalizePin_(student.pin) !== normalizePin_(pin)) {
      return failure_('รหัสนักเรียนหรือ PIN ไม่ถูกต้อง');
    }

    return success_('เข้าสู่ระบบสำเร็จ', {
      id: student.id,
      name: student.firstname,
      nickname: student.nickname,
      score: Number(student.examscore) || 0,
      attendance: calculateStudentAttendance_(student)
    });
  } catch (error) {
    return failure_(error.message);
  }
}

function getStudentPortal(studentId, pin) {
  return checkStudentLogin(studentId, pin);
}

function teacherLogin(username, password) {
  try {
    const sheet = getSheetSafe_(CONFIG.SHEETS.SETTINGS);
    const credentials = sheet.getRange('D2:E2').getDisplayValues()[0];
    const storedUsername = String(credentials[0] || '').trim();
    const storedPassword = String(credentials[1] || '').trim();
    const inputUsername = String(username || '').trim();
    const inputPassword = String(password || '').trim();

    if (inputUsername === storedUsername && inputPassword === storedPassword) {
      return success_('เข้าสู่ระบบครูสำเร็จ', { role: 'teacher' });
    }

    return failure_('Username หรือ Password ไม่ถูกต้อง');
  } catch (error) {
    return failure_(error.message);
  }
}

function getAttendance(date) {
  try {
    const sheet = getSheetSafe_(CONFIG.SHEETS.ATTENDANCE);
    const lastRow = sheet.getLastRow();
    const lastCol = sheet.getLastColumn();

    if (lastRow < 2 || lastCol < 3) {
      return { success: false, message: 'ยังไม่ได้สร้างปฏิทิน' };
    }

    const columnIndex = findAttendanceColumn_(sheet, date);
    if (columnIndex < 0) {
      return { success: false, message: 'ไม่พบวันที่ ' + getShortDate_(date) };
    }

    const ids = sheet.getRange(2, 1, lastRow - 1, 1).getDisplayValues().flat();
    const values = sheet.getRange(2, columnIndex + 1, lastRow - 1, 1).getValues().flat();
    const result = {};

    ids.forEach(function(id, index) {
      const key = normalizeId_(id);
      if (key !== '') result[key] = isPresent_(values[index]);
    });

    return {
      success: true,
      data: result,
      dateText: getShortDate_(date),
      dashboard: buildDashboardFromMap_(result)
    };
  } catch (error) {
    return { success: false, message: error.message };
  }
}

function buildDashboardFromMap_(attendanceMap) {
  const values = Object.keys(attendanceMap || {}).map(function(key) {
    return attendanceMap[key] === true;
  });
  const present = values.filter(Boolean).length;
  return { total: values.length, present: present, absent: values.length - present };
}

function getTodayDashboard(date) {
  const result = getAttendance(date);
  return result.success ? result.dashboard : { total: 0, present: 0, absent: 0 };
}

function saveAttendance(date, attendanceData) {
  try {
    if (!Array.isArray(attendanceData)) {
      return { success: false, message: 'รูปแบบข้อมูลการเช็คชื่อไม่ถูกต้อง' };
    }

    const sheet = getSheetSafe_(CONFIG.SHEETS.ATTENDANCE);
    const lastRow = sheet.getLastRow();
    const lastCol = sheet.getLastColumn();

    if (lastRow < 2 || lastCol < 3) {
      return { success: false, message: "กรุณากด 'สร้างปฏิทิน' ก่อนใช้งาน" };
    }

    const columnIndex = findAttendanceColumn_(sheet, date);
    if (columnIndex < 0) {
      return { success: false, message: 'ไม่พบวันที่ ' + getShortDate_(date) + ' ในตาราง' };
    }

    const ids = sheet.getRange(2, 1, lastRow - 1, 1).getDisplayValues().flat();
    const rowMap = {};
    ids.forEach(function(id, index) {
      const key = normalizeId_(id);
      if (key !== '') rowMap[key] = index;
    });

    const values = sheet.getRange(2, columnIndex + 1, lastRow - 1, 1).getValues();

    attendanceData.forEach(function(item) {
      if (!item || item.id == null) return;
      const rowIndex = rowMap[normalizeId_(item.id)];
      if (rowIndex !== undefined) values[rowIndex][0] = item.present === true;
    });

    sheet.getRange(2, columnIndex + 1, values.length, 1).setValues(values);
    calculateResult();

    return {
      success: true,
      message: 'บันทึกเรียบร้อยสำหรับวันที่ ' + getShortDate_(date),
      dashboard: getTodayDashboard(date)
    };
  } catch (error) {
    return { success: false, message: 'เกิดข้อผิดพลาด: ' + error.message };
  }
}

function generateCalendar() {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const settings = getSheetSafe_(CONFIG.SHEETS.SETTINGS);
    const attendance = getSheetSafe_(CONFIG.SHEETS.ATTENDANCE);
    const students = getSheetSafe_(CONFIG.SHEETS.STUDENTS);
    const holiday = ss.getSheetByName(CONFIG.SHEETS.HOLIDAY);

    const settingData = settings.getRange(1, 1, settings.getLastRow(), 2).getValues();
    let startDate = null;
    let endDate = null;

    settingData.forEach(function(row) {
      const key = String(row[0]).trim();
      if (key === 'StartDate') startDate = parseDate_(row[1]);
      if (key === 'EndDate') endDate = parseDate_(row[1]);
    });

    if (!startDate || !endDate) throw new Error('กรุณาตรวจสอบ StartDate และ EndDate');
    if (startDate > endDate) throw new Error('StartDate ต้องไม่มากกว่า EndDate');

    const holidayKeys = [];
    if (holiday && holiday.getLastRow() > 1) {
      holiday.getRange(2, 1, holiday.getLastRow() - 1, 1).getValues().flat()
        .forEach(function(value) {
          try { holidayKeys.push(getDateKey_(value)); } catch (error) {}
        });
    }

    const headers = ['ID', 'Nickname'];
    const current = new Date(startDate);

    while (current <= endDate) {
      const day = current.getDay();
      const dateKey = getDateKey_(current);
      if (day !== 0 && day !== 6 && !holidayKeys.includes(dateKey)) {
        headers.push(getShortDate_(current));
      }
      current.setDate(current.getDate() + 1);
    }

    attendance.clear();
    attendance.getRange(1, 1, 1, headers.length).setNumberFormat('@').setValues([headers]);

    const lastStudentRow = students.getLastRow();
    if (lastStudentRow > 1) {
      const studentData = students.getRange(2, 1, lastStudentRow - 1, 4).getValues();
      const output = studentData.map(function(row) { return [row[0], row[3]]; });
      attendance.getRange(2, 1, output.length, 2).setValues(output);
    }

    return 'สร้างปฏิทินเรียบร้อยแล้ว!';
  } catch (error) {
    return 'เกิดข้อผิดพลาด: ' + error.message;
  }
}

function calculateResult() {
  try {
    const students = getSheetSafe_(CONFIG.SHEETS.STUDENTS);
    const attendance = getSheetSafe_(CONFIG.SHEETS.ATTENDANCE);
    const studentLastRow = students.getLastRow();
    const attendanceLastRow = attendance.getLastRow();
    const attendanceLastCol = attendance.getLastColumn();

    if (studentLastRow < 2) return 'ไม่มีข้อมูลนักเรียน';
    const studentCount = studentLastRow - 1;

    if (attendanceLastRow < 2 || attendanceLastCol < 3) {
      const emptyResult = Array.from({ length: studentCount }, function() {
        return [0, 0, '0%', 0, 'ไม่ผ่าน'];
      });
      students.getRange(2, CONFIG.STUDENT_COLUMNS.PRESENT, studentCount, 5).setValues(emptyResult);
      return 'ยังไม่มีข้อมูลการเช็คชื่อ';
    }

    const attendanceCount = Math.min(studentCount, attendanceLastRow - 1);
    const attendanceData = attendance.getRange(2, 3, attendanceCount, attendanceLastCol - 2).getValues();
    const examData = students.getRange(2, CONFIG.STUDENT_COLUMNS.EXAM_SCORE, studentCount, 1).getValues();

    const finalData = Array.from({ length: studentCount }, function(_, index) {
      const row = attendanceData[index] || [];
      let present = 0;
      let checked = 0;

      row.forEach(function(value) {
        if (value !== '' && value !== null) {
          checked++;
          if (isPresent_(value)) present++;
        }
      });

      const absent = checked - present;
      const percent = checked > 0 ? Math.round((present / checked) * 100) : 0;
      const exam = Number(examData[index][0]) || 0;
      const result = percent >= CONFIG.PASS_RULES.ATTENDANCE_PERCENT &&
        exam >= CONFIG.PASS_RULES.EXAM_SCORE ? 'ผ่าน' : 'ไม่ผ่าน';

      return [present, absent, percent + '%', exam, result];
    });

    students.getRange(2, CONFIG.STUDENT_COLUMNS.PRESENT, finalData.length, 5).setValues(finalData);
    return 'คำนวณผลเรียบร้อย!';
  } catch (error) {
    return 'เกิดข้อผิดพลาด: ' + error.message;
  }
}

function getSummaryAll() {
  try {
    const sheet = getSheetSafe_(CONFIG.SHEETS.ATTENDANCE);
    const lastRow = sheet.getLastRow();
    const lastCol = sheet.getLastColumn();

    if (lastRow < 2 || lastCol < 3) {
      return { success: false, message: 'ยังไม่มีข้อมูลการเช็คชื่อ' };
    }

    const ids = sheet.getRange(2, 1, lastRow - 1, 1).getDisplayValues().flat();
    const nicknames = sheet.getRange(2, 2, lastRow - 1, 1).getDisplayValues().flat();
    const data = sheet.getRange(2, 3, lastRow - 1, lastCol - 2).getValues();
    const totalDays = lastCol - 2;

    const summary = ids.map(function(id, index) {
      let presentCount = 0;
      let checkedCount = 0;

      (data[index] || []).forEach(function(value) {
        if (value !== '' && value !== null) {
          checkedCount++;
          if (isPresent_(value)) presentCount++;
        }
      });

      const absentCount = checkedCount - presentCount;
      const percent = checkedCount > 0 ? Math.round((presentCount / checkedCount) * 100) : 0;

      return {
        id: id,
        nickname: nicknames[index],
        present: presentCount,
        absent: absentCount,
        percent: percent + '%',
        checked: checkedCount,
        totalDays: totalDays,
        remaining: Math.max(0, totalDays - checkedCount)
      };
    });

    return { success: true, summary: summary };
  } catch (error) {
    return { success: false, message: error.message };
  }
}

function getClassStatistics() {
  try {
    const sheet = getSheetSafe_(CONFIG.SHEETS.STUDENTS);
    const lastRow = sheet.getLastRow();

    if (lastRow < 2) {
      return success_('ไม่มีข้อมูล', {
        total: 0, pass: 0, fail: 0, avgAttendance: 0, avgExam: 0
      });
    }

    const data = sheet.getRange(2, CONFIG.STUDENT_COLUMNS.PRESENT, lastRow - 1, 5).getValues();
    let pass = 0;
    let fail = 0;
    let attendanceSum = 0;
    let examSum = 0;

    data.forEach(function(row) {
      const attendance = parseInt(String(row[2]).replace('%', ''), 10) || 0;
      const exam = Number(row[3]) || 0;
      attendanceSum += attendance;
      examSum += exam;
      if (String(row[4]).trim() === 'ผ่าน') pass++; else fail++;
    });

    const total = lastRow - 1;
    return success_('OK', {
      total: total,
      pass: pass,
      fail: fail,
      avgAttendance: Math.round(attendanceSum / total),
      avgExam: Math.round(examSum / total)
    });
  } catch (error) {
    return failure_(error.message);
  }
}

function getAttendanceChartData() {
  return getStudents().map(function(student) {
    return {
      name: student.nickname || student.firstname || student.id,
      attendance: parseInt(String(student.attendance || '0').replace('%', ''), 10) || 0
    };
  });
}

function getExamChartData() {
  return getStudents().map(function(student) {
    return {
      name: student.nickname || student.firstname || student.id,
      score: Number(student.examscore) || 0
    };
  });
}

function getTeacherDashboard(date) {
  return {
    daily: getTodayDashboard(date || new Date()),
    summary: getClassStatistics(),
    attendanceChart: getAttendanceChartData(),
    examChart: getExamChartData()
  };
}


/******************************************************
 * 13) Student Portal URL / QR
 ******************************************************/
function getStudentPortalUrl() {
  try {
    const baseUrl = ScriptApp.getService().getUrl();

    if (!baseUrl) {
      return failure_(
        'ยังไม่พบ URL ของ Web App กรุณา Deploy ระบบก่อน'
      );
    }

    return success_('OK', {
      url: baseUrl + '?page=Student'
    });

  } catch (error) {
    return failure_(error.message);
  }
}
