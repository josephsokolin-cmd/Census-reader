# Census-reader<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Resident Report Generator</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
    }
  </style>
</head>
<body>
  <div id="root"></div>

  <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

  <script type="text/babel">
    const { useState } = React;

    const Upload = () => (
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
        <polyline points="17 8 12 3 7 8"></polyline>
        <line x1="12" y1="3" x2="12" y2="15"></line>
      </svg>
    );

    const FileText = () => (
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
        <polyline points="14 2 14 8 20 8"></polyline>
        <line x1="16" y1="13" x2="8" y2="13"></line>
        <line x1="16" y1="17" x2="8" y2="17"></line>
        <polyline points="10 9 9 9 8 9"></polyline>
      </svg>
    );

    const Download = () => (
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
        <polyline points="7 10 12 15 17 10"></polyline>
        <line x1="12" y1="15" x2="12" y2="3"></line>
      </svg>
    );

    const AlertCircle = () => (
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <circle cx="12" cy="12" r="10"></circle>
        <line x1="12" y1="8" x2="12" y2="12"></line>
        <line x1="12" y1="16" x2="12.01" y2="16"></line>
      </svg>
    );

    const CheckCircle = () => (
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
        <path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"></path>
        <polyline points="22 4 12 14.01 9 11.01"></polyline>
      </svg>
    );

    const ResidentReportGenerator = () => {
      const [excelFile, setExcelFile] = useState(null);
      const [censusFile, setCensusFile] = useState(null);
      const [nextVisitDate, setNextVisitDate] = useState('');
      const [processing, setProcessing] = useState(false);
      const [logs, setLogs] = useState([]);
      const [result, setResult] = useState(null);
      const [error, setError] = useState(null);
      const [librariesLoaded, setLibrariesLoaded] = useState(false);

      const addLog = (message, type = 'info') => {
        setLogs(prev => [...prev, { message, type, timestamp: new Date().toISOString() }]);
      };

      const normalizeName = (name) => {
        if (!name) return '';
        
        let cleaned = String(name)
          .replace(/^\s*(Active|Status|Inactive)\s+/i, '')
          .trim()
          .toLowerCase();
        
        cleaned = cleaned
          .replace(/\s+jr\.?$/i, '')
          .replace(/\s+sr\.?$/i, '')
          .replace(/\s+iii$/i, '')
          .replace(/\s+ii$/i, '');
        
        cleaned = cleaned.replace(/[^a-z\s,]/g, '');
        
        if (cleaned.includes(',')) {
          const parts = cleaned.split(',').map(p => p.trim());
          if (parts.length === 2) {
            cleaned = `${parts[1]} ${parts[0]}`;
          }
        }
        
        cleaned = cleaned.replace(/\s+/g, ' ').trim();
        
        let normalized = cleaned;
        normalized = normalized.replace(/\bi\b/g, 'l');
        normalized = normalized.replace(/\bil/g, 'll');
        
        const parts = normalized.split(' ').filter(p => p.length > 0);
        return parts.sort().join(' ');
      };

      const loadLibraries = async () => {
        if (librariesLoaded) return;
        
        try {
          if (!window.XLSX) {
            addLog('Loading Excel library...');
            await new Promise((resolve, reject) => {
              const script = document.createElement('script');
              script.src = 'https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js';
              script.onload = resolve;
              script.onerror = () => reject(new Error('Failed to load XLSX library. Please check your internet connection.'));
              document.head.appendChild(script);
            });
          }
          
          if (!window.pdfjsLib) {
            addLog('Loading PDF library...');
            await new Promise((resolve, reject) => {
              const script = document.createElement('script');
              script.src = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js';
              script.onload = () => {
                window.pdfjsLib.GlobalWorkerOptions.workerSrc = 
                  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
                resolve();
              };
              script.onerror = () => reject(new Error('Failed to load PDF library. Please check your internet connection.'));
              document.head.appendChild(script);
            });
          }
          
          if (!window.mammoth) {
            addLog('Loading Word document library...');
            await new Promise((resolve, reject) => {
              const script = document.createElement('script');
              script.src = 'https://cdnjs.cloudflare.com/ajax/libs/mammoth/1.6.0/mammoth.browser.min.js';
              script.onload = resolve;
              script.onerror = () => reject(new Error('Failed to load Word library. Please check your internet connection.'));
              document.head.appendChild(script);
            });
          }
          
          setLibrariesLoaded(true);
          addLog('All libraries loaded successfully', 'success');
        } catch (err) {
          throw new Error(`Library loading failed: ${err.message}. Make sure you have internet connection.`);
        }
      };

      const parseDate = (dateStr) => {
        if (!dateStr) return null;
        const str = String(dateStr);
        const parts = str.split('/');
        if (parts.length === 3) {
          const month = parseInt(parts[0]);
          const day = parseInt(parts[1]);
          let year = parseInt(parts[2]);
          if (year < 100) year += 2000;
          return new Date(year, month - 1, day);
        }
        return new Date(dateStr);
      };

      const calculateNextDOS = (date) => {
        if (!date) return '';
        const nextDate = new Date(date);
        nextDate.setDate(nextDate.getDate() + 61);
        return `${(nextDate.getMonth() + 1).toString().padStart(2, '0')}/${nextDate.getDate().toString().padStart(2, '0')}/${nextDate.getFullYear().toString().slice(-2)}`;
      };

      const parseExcelData = (excelData) => {
        const headers = excelData[0];
        const rows = excelData.slice(1).filter(row => row && row.length > 0);
        
        const colMap = {
          template: headers.indexOf('Template'),
          provider: headers.indexOf('Provider'),
          patient: headers.indexOf('Patient'),
          facility: headers.indexOf('Facility'),
          visitDate: headers.indexOf('Visit Date'),
          cpts: headers.indexOf('CPTs'),
          icds: headers.indexOf('ICDs')
        };

        const facility = rows.length > 0 ? String(rows[0][colMap.facility] || '') : '';
        const doctorName = rows.length > 0 ? String(rows[0][colMap.provider] || '') : '';
        
        addLog(`Facility: "${facility}", Doctor: "${doctorName}"`);

        const patientMap = new Map();
        
        rows.forEach((row) => {
          const patientName = row[colMap.patient];
          const visitDateStr = row[colMap.visitDate];
          
          if (!patientName) return;

          const visitDate = parseDate(visitDateStr);
          const template = String(row[colMap.template] || '');
          let cpts = String(row[colMap.cpts] || '');
          
          if (template.includes('Patient Refusal Note')) {
            cpts = 'Patient Refusal Note';
          } else if (template.includes('Patient Not Found Note')) {
            cpts = 'Patient Not Found Note';
          }

          const patientData = {
            name: String(patientName),
            visitDate: visitDate,
            visitDateStr: String(visitDateStr),
            cpts: cpts,
            icds: String(row[colMap.icds] || ''),
            isNotFound: template.includes('Patient Not Found Note')
          };

          const normalizedName = normalizeName(patientName);
          if (!patientMap.has(normalizedName) || visitDate > patientMap.get(normalizedName).visitDate) {
            patientMap.set(normalizedName, patientData);
          }
        });

        addLog(`Processed ${patientMap.size} unique patients`, 'success');
        return { facility, doctorName, patientMap };
      };

      const parseExcelCensus = (data) => {
        const residents = [];
        const seenResidents = new Set();
        
        let roomColIndex = -1;
        let nameColIndex = -1;
        
        for (let i = 0; i < Math.min(10, data.length); i++) {
          const row = data[i];
          if (!row) continue;
          
          for (let j = 0; j < row.length; j++) {
            const cell = String(row[j] || '').toLowerCase().trim();
            if (cell.includes('room') && roomColIndex === -1) roomColIndex = j;
            if ((cell.includes('resident') || cell.includes('name')) && nameColIndex === -1) nameColIndex = j;
          }
        }
        
        let startRow = (roomColIndex !== -1 || nameColIndex !== -1) ? 1 : 0;
        
        for (let i = startRow; i < data.length; i++) {
          const row = data[i];
          if (!row || row.length === 0) continue;
          
          const possibleCombos = [
            [0, 1],
            [roomColIndex, nameColIndex],
            [1, 0],
            [0, 2],
            [1, 2],
          ].filter(combo => combo[0] !== -1 && combo[1] !== -1);
          
          let foundResident = false;
          
          for (const [roomIdx, nameIdx] of possibleCombos) {
            if (foundResident) break;
            if (roomIdx >= row.length || nameIdx >= row.length) continue;
            
            let roomCell = String(row[roomIdx] || '').trim().replace(/[\r\n]+/g, ' ').replace(/\s+/g, ' ');
            let nameCell = String(row[nameIdx] || '').trim().replace(/[\r\n]+/g, ' ').replace(/\s+/g, ' ');
            
            if (!roomCell || !nameCell) continue;
            if (roomCell.toLowerCase().includes('room') || roomCell.toLowerCase().includes('unit')) continue;
            if (nameCell.toLowerCase().includes('resident') || nameCell.toLowerCase().includes('name')) continue;
            if (nameCell === 'Empty' || nameCell === 'Vacant') continue;
            
            const roomMatch = roomCell.match(/(\d{2,4}[-]?[A-DP]?)/i);
            
            if (roomMatch) {
              const room = roomMatch[1].toUpperCase();
              const nameParts = nameCell.split(/[;\n]|(?:\s{3,})/);
              
              for (const part of nameParts) {
                const cleaned = part.trim();
                if (!cleaned || cleaned.length < 3) continue;
                
                let nameMatch = cleaned.match(/^([A-Z][a-zA-Z\s'-]+(?:\s+(?:Jr\.?|Sr\.?|II|III|IV))?),\s*([A-Z][a-zA-Z\s'-]+)/);
                
                if (nameMatch) {
                  const name = `${nameMatch[1].trim()}, ${nameMatch[2].trim()}`;
                  const key = `${room}:${name.toLowerCase()}`;
                  
                  if (!seenResidents.has(key)) {
                    seenResidents.add(key);
                    residents.push({ name, room });
                    foundResident = true;
                  }
                } else {
                  nameMatch = cleaned.match(/^([A-Z][a-z]+)\s+([A-Z][a-zA-Z\s'-]+?)$/);
                  if (nameMatch) {
                    const name = `${nameMatch[2].trim()}, ${nameMatch[1].trim()}`;
                    const key = `${room}:${name.toLowerCase()}`;
                    
                    if (!seenResidents.has(key)) {
                      seenResidents.add(key);
                      residents.push({ name, room });
                      foundResident = true;
                    }
                  }
                }
              }
            }
          }
        }
        
        return residents;
      };

      const parsePDFCensus = async (pdfArrayBuffer) => {
        const loadingTask = window.pdfjsLib.getDocument({ data: pdfArrayBuffer });
        const pdf = await loadingTask.promise;
        
        addLog(`PDF loaded: ${pdf.numPages} pages`, 'success');
        
        const allTextItems = [];
        for (let i = 1; i <= pdf.numPages; i++) {
          const page = await pdf.getPage(i);
          const textContent = await page.getTextContent();
          allTextItems.push(...textContent.items);
        }

        addLog(`Extracted ${allTextItems.length} text items from PDF`);

        if (allTextItems.length === 0) {
          throw new Error('PDF appears to be image-based. Please provide a text-based PDF or try Excel format.');
        }
        
        const residents = [];
        const seenResidents = new Set();
        
        addLog('Parsing PDF content...');
        
        const roomPositions = [];
        for (let i = 0; i < allTextItems.length; i++) {
          const item = allTextItems[i].str.trim();
          const roomMatch = item.match(/^(\d{2,4}[-]?[A-DP]?)$/i);
          if (roomMatch) {
            roomPositions.push({ room: roomMatch[1].toUpperCase(), index: i });
          }
        }
        
        addLog(`Found ${roomPositions.length} room numbers in PDF`);
        
        const skipPatterns = [
          /^(MLTSS|ICP|MCR|PVT|LTSS|HOS|INS|PA|PAP|MCD|MA|COM|SNF|ALF|IL)$/i,
          /^\([^)]+\)$/,
          /^(Active|Inactive|Status|Empty|Vacant)$/i,
          /^\d{1,2}\/\d{1,2}\/\d{2,4}$/,
          /^[\d\-\(\)\s]+$/
        ];
        
        for (const roomInfo of roomPositions) {
          const room = roomInfo.room;
          const startIdx = roomInfo.index;
          
          for (let j = startIdx + 1; j < Math.min(startIdx + 30, allTextItems.length); j++) {
            const text = allTextItems[j].str.trim();
            
            if (/^\d{2,4}[-]?[A-DP]?$/i.test(text)) break;
            
            if (skipPatterns.some(pattern => pattern.test(text))) continue;
            
            let nameMatch = text.match(/^([A-Z][a-z]+(?:\s+[A-Z][a-z]+)+?)(?:\s+\([^)]+\))?$/);
            if (nameMatch) {
              const nameParts = nameMatch[1].trim().split(/\s+/);
              if (nameParts.length >= 2) {
                const lastName = nameParts[nameParts.length - 1];
                const firstName = nameParts.slice(0, -1).join(' ');
                const fullName = `${lastName}, ${firstName}`;
                const key = `${room}:${fullName.toLowerCase()}`;
                
                if (!seenResidents.has(key)) {
                  seenResidents.add(key);
                  residents.push({ name: fullName, room: room });
                }
                break;
              }
            }
            
            nameMatch = text.match(/^([A-Z][a-zA-Z\s'-]+(?:\s+(?:Jr\.?|Sr\.?|II|III|IV))?),\s*([A-Z][a-zA-Z\s'-.]+)/i);
            if (nameMatch) {
              const fullName = `${nameMatch[1].trim()}, ${nameMatch[2].trim()}`;
              const key = `${room}:${fullName.toLowerCase()}`;
              
              if (!seenResidents.has(key)) {
                seenResidents.add(key);
                residents.push({ name: fullName, room: room });
              }
              break;
            }
          }
        }
        
        return residents;
      };

      const parseWordCensus = async (arrayBuffer) => {
        const textResult = await window.mammoth.extractRawText({ arrayBuffer });
        const text = textResult.value;
        
        addLog('Extracted text from Word document');
        
        const residents = [];
        const seenResidents = new Set();
        
        let pattern = /(\d{2,4}[-]?[A-DP]?)\s+([A-Z][a-zA-Z\s'-]+(?:\s+(?:Jr\.?|Sr\.?|II|III|IV))?),\s+([A-Z][a-zA-Z\s'-]+)/g;
        
        let match;
        while ((match = pattern.exec(text)) !== null) {
          const room = match[1].toUpperCase();
          const name = `${match[2].trim()}, ${match[3].trim()}`;
          const key = `${room}:${name.toLowerCase()}`;
          
          if (!seenResidents.has(key)) {
            seenResidents.add(key);
            residents.push({ name, room });
          }
        }
        
        return residents;
      };

      const processCensusFile = async () => {
        const fileName = censusFile.name.toLowerCase();
        
        if (fileName.endsWith('.xlsx') || fileName.endsWith('.xls')) {
          addLog('Reading Excel census file...');
          const arrayBuffer = await censusFile.arrayBuffer();
          const workbook = window.XLSX.read(arrayBuffer);
          
          let allResidents = [];
          workbook.SheetNames.forEach(sheetName => {
            const skipNames = ['summary', 'notes', 'archive', 'info'];
            if (skipNames.some(skip => sheetName.toLowerCase().includes(skip))) return;
            
            const worksheet = workbook.Sheets[sheetName];
            const data = window.XLSX.utils.sheet_to_json(worksheet, { header: 1 });
            const residents = parseExcelCensus(data);
            allResidents.push(...residents);
          });
          
          return allResidents;
          
        } else if (fileName.endsWith('.pdf')) {
          addLog('Reading PDF census file...');
          const arrayBuffer = await censusFile.arrayBuffer();
          return await parsePDFCensus(arrayBuffer);
          
        } else if (fileName.endsWith('.docx') || fileName.endsWith('.doc')) {
          addLog('Reading Word document census file...');
          const arrayBuffer = await censusFile.arrayBuffer();
          return await parseWordCensus(arrayBuffer);
          
        } else {
          throw new Error('Census file must be PDF, Word, or Excel');
        }
      };

      const processFiles = async () => {
        if (!excelFile || !censusFile) {
          setError('Please upload both files');
          return;
        }

        setProcessing(true);
        setLogs([]);
        setError(null);
        setResult(null);

        try {
          await loadLibraries();
          
          addLog('Reading Excel file...');
          const excelArrayBuffer = await excelFile.arrayBuffer();
          const workbook = window.XLSX.read(excelArrayBuffer);
          const worksheet = workbook.Sheets[workbook.SheetNames[0]];
          const excelData = window.XLSX.utils.sheet_to_json(worksheet, { header: 1 });
          
          const { facility, doctorName, patientMap } = parseExcelData(excelData);
          
          const censusResidents = await processCensusFile();
          addLog(`Found ${censusResidents.length} residents in census`, 'success');

          if (censusResidents.length === 0) {
            throw new Error('No residents found in census file. Please check the file format.');
          }

          const reportData = [];
          let matchCount = 0;

          censusResidents.forEach(censusResident => {
            const normalizedCensusName = normalizeName(censusResident.name);
            const matchedPatient = patientMap.get(normalizedCensusName);
            
            if (matchedPatient) {
              matchCount++;
              reportData.push({
                roomNumber: censusResident.room,
                residentName: censusResident.name,
                visitDate: matchedPatient.visitDateStr,
                nextDOS: calculateNextDOS(matchedPatient.visitDate),
                cpts: matchedPatient.cpts,
                icds: matchedPatient.icds,
                history: '',
                notes: ''
              });
            } else {
              reportData.push({
                roomNumber: censusResident.room,
                residentName: censusResident.name,
                visitDate: '',
                nextDOS: '',
                cpts: '',
                icds: '',
                history: 'Not Seen',
                notes: ''
              });
            }
          });

          addLog(`Matched ${matchCount} residents`, 'success');
          
          let nextExpectedVisit = nextVisitDate.trim();
          if (!nextExpectedVisit) {
            let latestVisit = null;
            patientMap.forEach(patient => {
              if (!latestVisit || patient.visitDate > latestVisit) {
                latestVisit = patient.visitDate;
              }
            });
            nextExpectedVisit = latestVisit ? calculateNextDOS(latestVisit) : '';
          }

          addLog('Generating Excel report...');
          const outputWorkbook = window.XLSX.utils.book_new();
          
          const wsData = [
            [`Facility: ${facility}`],
            [`Doctor: ${doctorName}`],
            [`Next Expected Visit: ${nextExpectedVisit}`],
            [],
            ['Room Number', 'Resident Name', 'Visit Date', 'Next DOS', 'CPT Codes', 'ICD Codes', 'History', 'Notes']
          ];

          reportData.forEach(row => {
            wsData.push([
              row.roomNumber,
              row.residentName,
              row.visitDate,
              row.nextDOS,
              row.cpts,
              row.icds,
              row.history,
              row.notes
            ]);
          });

          const ws = window.XLSX.utils.aoa_to_sheet(wsData);
          ws['!cols'] = [
            { wch: 12 }, { wch: 25 }, { wch: 12 }, { wch: 12 },
            { wch: 30 }, { wch: 40 }, { wch: 12 }, { wch: 30 }
          ];

          window.XLSX.utils.book_append_sheet(outputWorkbook, ws, 'Resident Report');
          const wbout = window.XLSX.write(outputWorkbook, { bookType: 'xlsx', type: 'array' });
          const blob = new Blob([wbout], { type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' });
          
          const filename = `${facility.replace(/[^a-z0-9]/gi, '_')}_Report_${new Date().toISOString().split('T')[0]}.xlsx`;
          
          setResult({ blob, filename, rowCount: reportData.length, matchCount });
          addLog('Report generated successfully!', 'success');

        } catch (err) {
          setError(`Error: ${err.message}`);
          addLog(`Error: ${err.message}`, 'error');
          console.error('Full error:', err);
        } finally {
          setProcessing(false);
        }
      };

      const downloadReport = () => {
        if (!result) return;
        const url = URL.createObjectURL(result.blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = result.filename;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      };

      return (
        <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-8">
          <div className="max-w-4xl mx-auto">
            <div className="bg-white rounded-lg shadow-xl p-8">
              <h1 className="text-3xl font-bold text-gray-800 mb-2">
                Resident Report Generator
              </h1>
              <p className="text-sm text-gray-600 mb-6">
                Upload your Excel data and census file to generate a comprehensive resident report
              </p>

              <div className="space-y-6">
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">
                    Upload Excel File (Residents Seen)
                  </label>
                  <label className="flex-1 flex items-center justify-center px-4 py-3 border-2 border-dashed border-gray-300 rounded-lg cursor-pointer hover:border-indigo-500 transition">
                    <FileText />
                    <span className="text-sm text-gray-600 ml-2">
                      {excelFile ? excelFile.name : 'Choose Excel file...'}
                    </span>
                    <input
                      type="file"
                      accept=".xlsx,.xls"
                      onChange={(e) => setExcelFile(e.target.files[0])}
                      className="hidden"
                    />
                  </label>
                </div>

                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">
                    Upload Census File (PDF, Word, or Excel)
                  </label>
                  <label className="flex-1 flex items-center justify-center px-4 py-3 border-2 border-dashed border-gray-300 rounded-lg cursor-pointer hover:border-indigo-500 transition">
                    <Upload />
                    <span className="text-sm text-gray-600 ml-2">
                      {censusFile ? censusFile.name : 'Choose PDF, Word, or Excel file...'}
                    </span>
                    <input
                      type="file"
                      accept=".pdf,.docx,.doc,.xlsx,.xls"
                      onChange={(e) => setCensusFile(e.target.files[0])}
                      className="hidden"
                    />
                  </label>
                  <p className="text-xs text-gray-500 mt-1">
                    Excel format recommended for best accuracy
                  </p>
                </div>

                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">
                    Next Expected Visit Date (Optional)
                  </label>
                  <input
                    type="text"
                    placeholder="MM/DD/YY (leave blank to auto-calculate)"
                    value={nextVisitDate}
                    onChange={(e) => setNextVisitDate(e.target.value)}
                    className="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500"
                  />
                  <p className="text-xs text-gray-500 mt-1">
                    If blank, calculates as latest visit + 61 days
                  </p>
                </div>

                <button
                  onClick={processFiles}
                  disabled={!excelFile || !censusFile || processing}
                  className="w-full bg-indigo-600 text-white py-3 rounded-lg font-medium hover:bg-indigo-700 disabled:bg-gray-400 disabled:cursor-not-allowed transition"
                >
                  {processing ? 'Processing...' : 'Generate Report'}
                </button>

                {error && (
                  <div className="bg-red-50 border border-red-200 rounded-lg p-4 flex items-start gap-3">
                    <AlertCircle />
                    <div className="flex-1">
                      <p className="text-sm text-red-800 font-medium">Error</p>
                      <p className="text-sm text-red-700 mt-1">{error}</p>
                    </div>
                  </div>
                )}

                {result && (
                  <div className="bg-green-50 border border-green-200 rounded-lg p-4">
                    <div className="flex items-start gap-3 mb-4">
                      <CheckCircle />
                      <div>
                        <p className="text-sm font-medium text-green-800">
                          Report generated successfully!
                        </p>
                        <p className="text-sm text-green-700 mt-1">
                          {result.rowCount} residents processed, {result.matchCount} matched
                        </p>
                      </div>
                    </div>
                    <button
                      onClick={downloadReport}
                      className="w-full bg-green-600 text-white py-2 rounded-lg font-medium hover:bg-green-700 flex items-center justify-center gap-2 transition"
                    >
                      <Download />
                      Download Excel Report
                    </button>
                  </div>
                )}

                {logs.length > 0 && (
                  <div className="bg-gray-50 rounded-lg p-4 max-h-96 overflow-y-auto">
                    <h3 className="text-sm font-medium text-gray-700 mb-2">Processing Log:</h3>
                    <div className="space-y-1">
                      {logs.map((log, idx) => (
                        <div
                          key={idx}
                          className={`text-xs font-mono ${
                            log.type === 'error' ? 'text-red-600' :
                            log.type === 'success' ? 'text-green-600' :
                            'text-gray-600'
                          }`}
                        >
                          {log.message}
                        </div>
                      ))}
                    </div>
                  </div>
                )}
              </div>

              <div className="mt-8 pt-6 border-t border-gray-200">
                <p className="text-xs text-gray-500 text-center">
                  This tool requires an active internet connection to load processing libraries
                </p>
              </div>
            </div>
          </div>
        </div>
      );
    };

    ReactDOM.render(<ResidentReportGenerator />, document.getElementById('root'));
  </script>
</body>
</html>
