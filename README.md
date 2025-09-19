<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>OMR PDF K-Level Counter</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 40px; background: #e6f4ff; }
    h1 { text-align: center; color: #003366; }
    .college-header {
      display: flex; align-items: center; justify-content: center;
      background: #b3daff; padding: 15px; border-radius: 10px; margin-bottom: 20px;
    }
    .college-header img { height: 80px; margin-right: 20px; border-radius: 8px; }
    .college-header h2 { margin: 0; color: #003366; }
    .upload-box {
      text-align: center; margin: 20px auto; padding: 30px;
      border: 2px dashed #3399ff; border-radius: 10px; background: #cce6ff; width: 460px;
    }
    input[type="file"] { margin-top: 10px; }
    #status { text-align: center; margin: 20px; font-weight: bold; color: #003366; }
    #student-info, #results {
      background: #ffffff; padding: 15px; margin: 20px auto; width: 600px;
      border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,.1); font-size: 16px;
    }
    table { width: 100%; border-collapse: collapse; margin-top: 10px; }
    th, td { border: 1px solid #99ccff; padding: 8px; text-align: center; }
    th { background: #b3daff; }
    td { font-weight: bold; }
    ul { list-style-type: none; padding: 0; }
    ul li { margin: 5px 0; }
    .answered { color: green; font-weight: bold; }
    .unanswered { color: red; font-weight: bold; }
    .export-buttons { text-align: center; margin: 20px; }
    .export-buttons button {
      background: #007bff; color: white; border: none; padding: 10px 15px;
      margin: 5px; border-radius: 6px; cursor: pointer;
    }
    .export-buttons button:hover { background: #0056b3; }
  </style>
</head>
<body>
  <div class="college-header">
    <img src="college_token.avif" alt="College Logo">
    <div>
      <h2>Government Arts College, Nandanam, Chennai-35</h2>
    </div>
  </div>

  <h1>OMR PDF K-Level Counter</h1>

  <div class="upload-box">
    <p>Select a <b>.pdf</b> file to process:</p>
    <input type="file" id="fileInput" accept=".pdf">
  </div>

  <div id="status"></div>
  <div id="student-info"></div>
  <div id="results"></div>

  <div class="export-buttons" style="display:none;" id="exportSection">
    <button onclick="downloadCSV()">⬇ Download CSV</button>
    <button onclick="downloadPDF()">⬇ Download PDF</button>
  </div>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
  <script>
    let lastProcessedData = null;

    document.getElementById("fileInput").addEventListener("change", handleFile);

    async function handleFile(event) {
      const file = event.target.files[0];
      if (!file) return;

      setStatus("⏳ Processing file...");
      setResults("");
      setStudentInfo("");
      document.getElementById("exportSection").style.display = "none";

      try {
        const extractedText = await extractTextFromPDF(file);
        const { department, subjectCode, regNo, studentName, subjectName } = extractStudentInfo(extractedText);
        const { counts, total, unanswered, details } = countKLevels(extractedText);

        lastProcessedData = { department, subjectCode, regNo, studentName, subjectName, counts, total, unanswered, details };

        let studentHtml = "<h3>Student Information</h3><table>";
        studentHtml += `<tr><td><b>Student Name</b></td><td>${studentName || "Not Found"}</td></tr>`;
        studentHtml += `<tr><td><b>Register No</b></td><td>${regNo || "Not Found"}</td></tr>`;
        studentHtml += `<tr><td><b>Department</b></td><td>${department || "Not Found"}</td></tr>`;
        studentHtml += `<tr><td><b>Subject Name</b></td><td>${subjectName || "Not Found"}</td></tr>`;
        studentHtml += `<tr><td><b>Subject Code</b></td><td>${subjectCode || "Not Found"}</td></tr>`;
        studentHtml += "</table>";
        setStudentInfo(studentHtml);

        let html = "<h3>K-Level Counts</h3>";
        html += "<table><tr><th>K-Level</th><th>Count</th></tr>";
        for (const k of ["K1","K2","K3","K4","K5","K6"]) {
          html += `<tr><td>${k}</td><td>${counts[k]}</td></tr>`;
        }
        html += `<tr><td><b>Total Answered</b></td><td><b>${total}</b></td></tr>`;
        html += `<tr><td><b>Total Unanswered</b></td><td><b>${unanswered}</b></td></tr>`;
        html += `<tr><td><b>Total Questions</b></td><td><b>${details.length}</b></td></tr>`;
        html += "</table>";

        html += "<h3>Question-wise Answers</h3><ul>";
        details.forEach((d, idx) => {
          html += `<li>Q${idx+1}: ${d.pickedK ? `<span class="answered">${d.pickedK}</span>` : `<span class="unanswered">Unanswered ❌</span>`}</li>`;
        });
        html += "</ul>";

        setResults(html);
        setStatus("✅ Done");
        document.getElementById("exportSection").style.display = "block";
      } catch (err) {
        setStatus("❌ Error: " + err.message);
      }
    }

    function setStatus(msg) { document.getElementById("status").innerText = msg; }
    function setResults(html) { document.getElementById("results").innerHTML = html; }
    function setStudentInfo(html) { document.getElementById("student-info").innerHTML = html; }

    async function extractTextFromPDF(file) {
      const arrayBuffer = await file.arrayBuffer();
      const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;
      let text = "";
      for (let pageNum = 1; pageNum <= pdf.numPages; pageNum++) {
        const page = await pdf.getPage(pageNum);
        const content = await page.getTextContent();
        const pageText = content.items.map(item => item.str).join(" ");
        text += pageText + "\n";
      }
      text = text.replace(/\s+Q\.No\./gi, "\nQ.No.");
      text = text.replace(/(\d+)\s+\(/g, "\n$1 (");
      return text;
    }

    function extractStudentInfo(text) {
      text = text.replace(/\s+/g, " ");
      text = text.replace(/([A-Z])\s+([a-z]+)/g, "$1$2");
      text = text.replace(/Student\s*Name\s*:/i, "\nStudent Name: ");
      text = text.replace(/Register\s*No\s*:/i, "\nRegister No: ");
      text = text.replace(/Department\s*:/i, "\nDepartment: ");
      text = text.replace(/Subject\s*Name\s*:/i, "\nSubject Name: ");
      text = text.replace(/Subject\s*Code\s*:/i, "\nSubject Code: ");

      const nameMatch = text.match(/Student\s*Name[:.\- ]+([A-Za-z .]+)/i);
      const regMatch = text.match(/Register\s*No[:.\- ]+([A-Za-z0-9]+)/i);
      const deptMatch = text.match(/Department[:.\- ]+([A-Za-z0-9.\- ]+)/i);
      const subjNameMatch = text.match(/Subject\s*Name[:.\- ]+([A-Za-z0-9 .]+)/i);
      const subjCodeMatch = text.match(/Subject\s*Code[:.\- ]+([A-Za-z0-9]+)/i);

      return {
        studentName: nameMatch ? nameMatch[1].trim() : null,
        regNo: regMatch ? regMatch[1].trim() : null,
        department: deptMatch ? deptMatch[1].replace(/\s+/g, " ").trim() : null,
        subjectName: subjNameMatch ? subjNameMatch[1].trim() : null,
        subjectCode: subjCodeMatch ? subjCodeMatch[1].trim() : null
      };
    }

    function countKLevels(rawText) {
      const counts = { K1:0, K2:0, K3:0, K4:0, K5:0, K6:0 };
      const qRegex = /^(\d+)\s+\((.*?)\)\s+\((.*?)\)\s+\((.*?)\)\s+\((.*?)\)\s+\((.*?)\)\s+\((.*?)\)/gm;
      const details = [];

      let match;
      while ((match = qRegex.exec(rawText)) !== null) {
        const qNo = match[1];
        const choices = match.slice(2, 8);
        const pickedIndex = choices.findIndex(c => c.includes("●"));
        if (pickedIndex >= 0) counts[`K${pickedIndex+1}`]++;
        details.push({ q: qNo, pickedK: pickedIndex >= 0 ? `K${pickedIndex+1}` : null });
      }

      const total = Object.values(counts).reduce((a,b)=>a+b,0);
      const unanswered = details.filter(d => !d.pickedK).length;
      return { counts, total, unanswered, details };
    }

    function downloadCSV() {
      if (!lastProcessedData) return;
      let csv = "Question,Selected K-Level\n";
      lastProcessedData.details.forEach((d, idx) => {
        csv += `Q${idx+1},${d.pickedK || "Unanswered"}\n`;
      });
      const blob = new Blob([csv], { type: "text/csv" });
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = `${lastProcessedData.regNo || "student"}_results.csv`;
      link.click();
    }

    function downloadPDF() {
      if (!lastProcessedData) return;
      const { jsPDF } = window.jspdf;
      const doc = new jsPDF();
      doc.setFontSize(14);
      doc.text("OMR K-Level Report", 10, 10);
      doc.setFontSize(11);
      doc.text(`Student: ${lastProcessedData.studentName || "N/A"}`, 10, 20);
      doc.text(`Reg No: ${lastProcessedData.regNo || "N/A"}`, 10, 27);
      doc.text(`Department: ${lastProcessedData.department || "N/A"}`, 10, 34);
      doc.text(`Subject: ${lastProcessedData.subjectName || "N/A"} (${lastProcessedData.subjectCode || "N/A"})`, 10, 41);
      doc.text("K-Level Counts:", 10, 50);
      let y = 58;
      for (const k of ["K1","K2","K3","K4","K5","K6"]) {
        doc.text(`${k}: ${lastProcessedData.counts[k]}`, 10, y);
        y += 7;
      }
      doc.text(`Total Answered: ${lastProcessedData.total}`, 10, y); y+=7;
      doc.text(`Total Unanswered: ${lastProcessedData.unanswered}`, 10, y); y+=7;
      doc.text("Question-wise:", 10, y+5);
      let qy = y+12;
      lastProcessedData.details.forEach((d, idx) => {
        doc.text(`Q${idx+1}: ${d.pickedK || "Unanswered"}`, 10, qy);
        qy += 6;
        if (qy > 280) { doc.addPage(); qy = 20; }
      });
      doc.save(`${lastProcessedData.regNo || "student"}_results.pdf`);
    }
  </script>
</body>
</html>
