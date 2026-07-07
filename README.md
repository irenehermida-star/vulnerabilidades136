<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>RDR Migration Kanban - Java 8 → JDK 17</title>
  <style>
    * { box-sizing: border-box; }
    
    :root {
      --accent: #004481;
      --accent-soft: #5bbeff;
      --good: #008a67;
      --warn: #c47f00;
      --bad: #b92a45;
      --bg: #f0f3f7;
      --panel: #ffffff;
      --line: rgba(4, 50, 99, 0.1);
      --muted: #52697f;
      --ink: #061b34;
    }

    body {
      margin: 0;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
      background: var(--bg);
      color: var(--ink);
    }

    .header {
      background: linear-gradient(135deg, #004481, #1973b8);
      color: white;
      padding: 20px 24px;
      box-shadow: 0 4px 12px rgba(0, 68, 129, 0.2);
      position: sticky;
      top: 0;
      z-index: 100;
      display: flex;
      justify-content: space-between;
      align-items: center;
      flex-wrap: wrap;
      gap: 15px;
    }

    .header h1 {
      margin: 0 0 4px;
      font-size: 1.6rem;
      font-weight: 700;
    }

    .header p {
      margin: 0;
      font-size: 0.9rem;
      opacity: 0.9;
    }

    .header-actions {
      display: flex;
      gap: 10px;
      align-items: center;
      flex-wrap: wrap;
    }

    .status-indicator {
      display: inline-flex;
      align-items: center;
      gap: 8px;
      font-size: 0.85rem;
      padding: 6px 12px;
      border-radius: 99px;
      background: rgba(255, 255, 255, 0.15);
      border: 1px solid rgba(255, 255, 255, 0.2);
    }

    .status-dot {
      width: 10px;
      height: 10px;
      border-radius: 50%;
      background: #94a3b8;
    }

    .status-dot.syncing { background: #ff9500; animation: pulse 1s infinite alternate; }
    .status-dot.connected { background: #34c759; }
    .status-dot.disconnected { background: #ff3b30; }

    @keyframes pulse {
      from { opacity: 0.4; }
      to { opacity: 1; }
    }

    .btn {
      padding: 8px 16px;
      border: none;
      border-radius: 6px;
      font-size: 0.85rem;
      cursor: pointer;
      font-weight: 600;
      transition: all 150ms ease;
    }

    .btn-refresh {
      background: white;
      color: var(--accent);
    }

    .btn-refresh:hover {
      background: #f0f3f7;
    }

    /* Loading overlay */
    .loading-overlay {
      position: fixed;
      inset: 0;
      background: rgba(240, 243, 247, 0.95);
      z-index: 2000;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      gap: 20px;
      transition: opacity 0.3s ease;
    }

    .spinner {
      width: 45px;
      height: 45px;
      border: 3px solid rgba(0, 68, 129, 0.1);
      border-radius: 50%;
      border-top-color: var(--accent);
      animation: spin 1s linear infinite;
    }

    @keyframes spin {
      to { transform: rotate(360deg); }
    }

    .container {
      max-width: 1850px;
      margin: 0 auto;
      padding: 20px;
    }

    .filter-bar {
      margin-bottom: 20px;
      display: flex;
      gap: 12px;
      flex-wrap: wrap;
    }

    .filter-bar input, .filter-bar select {
      padding: 10px 14px;
      border: 1px solid var(--line);
      border-radius: 6px;
      background: white;
      color: var(--ink);
      font-size: 0.9rem;
      outline: none;
    }

    .filter-bar input::placeholder {
      color: var(--muted);
    }

    .filter-bar input:focus, .filter-bar select:focus {
      border-color: var(--accent);
    }

    .swimlane {
      margin-bottom: 24px;
      background: var(--panel);
      border-radius: 12px;
      overflow: hidden;
      box-shadow: 0 2px 8px rgba(4, 50, 99, 0.08);
      border: 1px solid var(--line);
    }

    .swimlane-header {
      background: linear-gradient(90deg, rgba(0, 68, 129, 0.04), transparent);
      padding: 16px 20px;
      border-bottom: 2px solid var(--line);
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .swimlane-title {
      font-size: 1.2rem;
      font-weight: 700;
      color: var(--accent);
      margin: 0;
    }

    .swimlane-info {
      font-size: 0.85rem;
      color: var(--muted);
      display: flex;
      gap: 12px;
      align-items: center;
    }

    .progress-indicator {
      width: 100px;
      height: 8px;
      background: rgba(4, 50, 99, 0.1);
      border-radius: 4px;
      overflow: hidden;
    }

    .progress-bar {
      height: 100%;
      background: linear-gradient(90deg, var(--good), var(--accent-soft));
      width: 0%;
      transition: width 300ms ease;
    }

    .columns-wrapper {
      display: flex;
      gap: 12px;
      padding: 16px;
      overflow-x: auto;
      background: var(--bg);
    }

    .column {
      flex: 0 0 270px;
      background: rgba(4, 50, 99, 0.04);
      border-radius: 8px;
      padding: 12px;
      border: 2px dashed var(--line);
      min-height: 400px;
      display: flex;
      flex-direction: column;
    }

    .column-header {
      font-weight: 700;
      font-size: 0.85rem;
      color: var(--ink);
      margin-bottom: 12px;
      padding-bottom: 8px;
      border-bottom: 2px solid var(--line);
      text-transform: uppercase;
      letter-spacing: 0.05em;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .column-count {
      background: rgba(4, 50, 99, 0.08);
      padding: 1px 6px;
      border-radius: 99px;
      font-size: 0.75rem;
      color: var(--muted);
    }

    .cards-container {
      flex: 1;
      display: flex;
      flex-direction: column;
      gap: 10px;
      min-height: 100px;
    }

    .card {
      background: white;
      border: 1px solid var(--line);
      border-radius: 8px;
      padding: 12px;
      cursor: move;
      transition: all 200ms ease;
      box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
      display: flex;
      flex-direction: column;
      gap: 8px;
    }

    .card:hover {
      box-shadow: 0 4px 12px rgba(4, 50, 99, 0.15);
      transform: translateY(-2px);
    }

    .card.dragging {
      opacity: 0.5;
    }

    .card-header {
      font-weight: 700;
      font-size: 0.9rem;
      color: var(--ink);
    }

    .card-risk {
      display: inline-block;
      padding: 3px 8px;
      border-radius: 999px;
      font-size: 0.75rem;
      font-weight: 600;
      text-transform: uppercase;
      align-self: flex-start;
    }

    .risk-high { background: rgba(185, 42, 69, 0.15); color: var(--bad); }
    .risk-medium { background: rgba(180, 127, 0, 0.15); color: var(--warn); }
    .risk-low { background: rgba(0, 138, 103, 0.15); color: var(--good); }

    .card-meta {
      font-size: 0.8rem;
      color: var(--muted);
      padding: 8px;
      background: rgba(4, 50, 99, 0.04);
      border-radius: 4px;
      line-height: 1.4;
      border: 1px solid rgba(4, 50, 99, 0.04);
    }

    .checklist {
      margin: 0;
      padding: 0;
      list-style: none;
      font-size: 0.8rem;
      border-top: 1px solid rgba(4, 50, 99, 0.06);
      padding-top: 8px;
    }

    .checklist-item {
      display: flex;
      align-items: center;
      padding: 5px 0;
      gap: 8px;
    }

    .checklist-item input[type="checkbox"] {
      cursor: pointer;
      width: 15px;
      height: 15px;
      accent-color: var(--good);
    }

    .checklist-item label {
      cursor: pointer;
      flex: 1;
      user-select: none;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .checklist-item input[type="checkbox"]:checked + label {
      color: var(--muted);
      text-decoration: line-through;
    }

    .column-empty {
      color: var(--muted);
      font-size: 0.85rem;
      text-align: center;
      padding: 20px 8px;
      opacity: 0.6;
    }

    .toast {
      position: fixed;
      bottom: 20px;
      right: 20px;
      background: var(--good);
      color: white;
      padding: 14px 20px;
      border-radius: 8px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
      animation: slideIn 300ms ease, slideOut 300ms ease 3700ms;
      z-index: 1000;
    }

    @keyframes slideIn {
      from { transform: translateX(400px); opacity: 0; }
      to { transform: translateX(0); opacity: 1; }
    }

    @keyframes slideOut {
      to { transform: translateX(400px); opacity: 0; }
    }
  </style>
</head>
<body>

  <!-- Loading Overlay -->
  <div class="loading-overlay" id="loadingOverlay">
    <div class="spinner"></div>
    <div style="font-size:16px; font-weight:600; color:var(--ink);" id="loadingText">Conectando con Google Sheets...</div>
  </div>

  <div class="header">
    <div class="header-info">
      <h1>🚀 RDR Migration Kanban - Java 8 → JDK 17</h1>
      <p>Tablero Kanban visual por procesos RDR. Arrastra las tarjetas para cambiar su estado.</p>
    </div>
    <div class="header-actions">
      <div class="status-indicator">
        <div class="status-dot" id="syncDot"></div>
        <span id="syncText">Cargando...</span>
      </div>
      <button id="btnRefresh" class="btn btn-refresh" title="Sincronizar ahora con Google Sheets">Sincronizar</button>
    </div>
  </div>

  <div class="container">
    <div class="filter-bar">
      <input type="text" id="searchFilter" placeholder="Buscar proceso o Java..." onkeyup="filterSwimlanes()">
      <select id="filterRisk" onchange="filterSwimlanes()">
        <option value="all">Todos los Riesgos</option>
        <option value="high">Riesgo Alto</option>
        <option value="medium">Riesgo Medio</option>
        <option value="low">Riesgo Bajo</option>
      </select>
    </div>

    <div id="swimlanes-container"></div>
  </div>

  <script>
    // Configuration - Paste your deployed Google Apps Script Web App URL here
    const SCRIPT_URL = "https://script.google.com/macros/s/AKfycbwWkm4M4wYf-oF-iVjD-b0rXyN4VyHWnU5Qx3wtel8YSo-DjO1q7cGIgAqNz2YuSn3v1Q/exec";

    // Embedded fallback projects database
    const initialProjects = [
  {
    "id": "DeltaSSIs",
    "name": "DeltaSSIs",
    "javaFiles": 11,
    "testFiles": 0,
    "hasPom": false,
    "score": 334.6,
    "process": "ALERT_FX_REPORT",
    "risk": "high",
    "riskLabel": "Riesgo Alto",
    "comment": "",
    "tech": {
      "jaxb": 83,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 51
    },
    "initialList": "initial"
  },
  {
    "id": "DeltaSSIsFX",
    "name": "DeltaSSIsFX",
    "javaFiles": 11,
    "testFiles": 0,
    "hasPom": false,
    "score": 334.6,
    "process": "ALERT_FX_REPORT",
    "risk": "high",
    "riskLabel": "Riesgo Alto",
    "comment": "",
    "tech": {
      "jaxb": 83,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 51
    },
    "initialList": "initial"
  },
  {
    "id": "DeltaSSIProject",
    "name": "DeltaSSIProject",
    "javaFiles": 11,
    "testFiles": 0,
    "hasPom": false,
    "score": 334.6,
    "process": "ALERT_FX_REPORT",
    "risk": "high",
    "riskLabel": "Riesgo Alto",
    "comment": "",
    "tech": {
      "jaxb": 83,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 51
    },
    "initialList": "initial"
  },
  {
    "id": "refinitivderivativesloader",
    "name": "refinitivderivativesloader",
    "javaFiles": 49,
    "testFiles": 0,
    "hasPom": true,
    "score": 306.2,
    "process": "Calypso-Filters",
    "risk": "high",
    "riskLabel": "Riesgo Alto",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 197,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "servicesRDR",
    "name": "servicesRDR",
    "javaFiles": 36,
    "testFiles": 0,
    "hasPom": false,
    "score": 221,
    "process": "Services-RDR",
    "risk": "high",
    "riskLabel": "Riesgo Alto",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 56,
      "sun": 1,
      "sec": 1,
      "xml": 73
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_SectorizacionEmisores",
    "name": "RDR_SectorizacionEmisores",
    "javaFiles": 28,
    "testFiles": 17,
    "hasPom": true,
    "score": 137.2,
    "process": "Services-RDR",
    "risk": "high",
    "riskLabel": "Riesgo Alto",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 28,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "sectorclassificationloader",
    "name": "sectorclassificationloader",
    "javaFiles": 24,
    "testFiles": 0,
    "hasPom": true,
    "score": 134.2,
    "process": "Calypso-Filters",
    "risk": "high",
    "riskLabel": "Riesgo Alto",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 82,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "rdrRules",
    "name": "rdrRules",
    "javaFiles": 74,
    "testFiles": 5,
    "hasPom": true,
    "score": 93.2,
    "process": "Rules-Engine",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 13
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Extraction_CPARTYS",
    "name": "RDR_Extraction_CPARTYS",
    "javaFiles": 18,
    "testFiles": 0,
    "hasPom": false,
    "score": 86.4,
    "process": "Services-RDR",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 231
    },
    "initialList": "initial"
  },
  {
    "id": "refinitivFilter",
    "name": "refinitivFilter",
    "javaFiles": 21,
    "testFiles": 0,
    "hasPom": true,
    "score": 84.7,
    "process": "Calypso-Filters",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 46,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "concurrenteRDR",
    "name": "concurrenteRDR",
    "javaFiles": 14,
    "testFiles": 0,
    "hasPom": false,
    "score": 77.2,
    "process": "Services-RDR",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 203
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_ConciliaColombia",
    "name": "RDR_ConciliaColombia",
    "javaFiles": 4,
    "testFiles": 0,
    "hasPom": false,
    "score": 75.8,
    "process": "Services-RDR",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 12,
      "jpa": 0,
      "jms": 0,
      "sun": 1,
      "sec": 0,
      "xml": 1
    },
    "initialList": "initial"
  },
  {
    "id": "RDRProcessFile",
    "name": "RDRProcessFile",
    "javaFiles": 54,
    "testFiles": 15,
    "hasPom": false,
    "score": 66.6,
    "process": "Services-RDR",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_ConTotal",
    "name": "RDR_ConTotal",
    "javaFiles": 46,
    "testFiles": 0,
    "hasPom": false,
    "score": 65.4,
    "process": "ConTotal",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "anna_download",
    "name": "anna_download",
    "javaFiles": 34,
    "testFiles": 0,
    "hasPom": true,
    "score": 59.2,
    "process": "Otros Proyectos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 6,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 8
    },
    "initialList": "initial"
  },
  {
    "id": "JSON_Altamira",
    "name": "JSON_Altamira",
    "javaFiles": 37,
    "testFiles": 0,
    "hasPom": false,
    "score": 57.3,
    "process": "Otros Proyectos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ProjectDAO",
    "name": "ProjectDAO",
    "javaFiles": 32,
    "testFiles": 0,
    "hasPom": true,
    "score": 54.3,
    "process": "Otros Proyectos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 15,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargadorEntity",
    "name": "RDR_CargadorEntity",
    "javaFiles": 28,
    "testFiles": 0,
    "hasPom": false,
    "score": 53.2,
    "process": "Services-RDR",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 1,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "CargaMentor_LA",
    "name": "CargaMentor_LA",
    "javaFiles": 24,
    "testFiles": 0,
    "hasPom": false,
    "score": 46.8,
    "process": "Otros Proyectos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 6
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Refinitiv_Request",
    "name": "RDR_Refinitiv_Request",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 38.5,
    "process": "Services-RDR",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 1,
      "sec": 0,
      "xml": 11
    },
    "initialList": "initial"
  },
  {
    "id": "LegalOpinion",
    "name": "LegalOpinion",
    "javaFiles": 16,
    "testFiles": 0,
    "hasPom": false,
    "score": 38.4,
    "process": "Otros Proyectos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "TSEC_Generator",
    "name": "TSEC_Generator",
    "javaFiles": 15,
    "testFiles": 0,
    "hasPom": false,
    "score": 37.5,
    "process": "Otros Proyectos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "Legal_Opinion_Cargador",
    "name": "Legal_Opinion_Cargador",
    "javaFiles": 15,
    "testFiles": 0,
    "hasPom": false,
    "score": 37.5,
    "process": "Otros Proyectos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "Investors_csv_process",
    "name": "Investors_csv_process",
    "javaFiles": 21,
    "testFiles": 1,
    "hasPom": false,
    "score": 36.9,
    "process": "Otros Proyectos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Borrado_Emisiones",
    "name": "RDR_Borrado_Emisiones",
    "javaFiles": 6,
    "testFiles": 0,
    "hasPom": false,
    "score": 36.6,
    "process": "Indices",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 2,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 1
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_MifidII",
    "name": "RDR_MifidII",
    "javaFiles": 13,
    "testFiles": 0,
    "hasPom": false,
    "score": 35.7,
    "process": "Mifid-II",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "AltaFondos_Genera_csv",
    "name": "AltaFondos_Genera_csv",
    "javaFiles": 8,
    "testFiles": 0,
    "hasPom": false,
    "score": 35.2,
    "process": "AltaFondos",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 1,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Emisiones_PLSQL",
    "name": "RDR_Emisiones_PLSQL",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 33.9,
    "process": "Indices",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 2,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 1
    },
    "initialList": "initial"
  },
  {
    "id": "javacsv",
    "name": "javacsv",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 33.8,
    "process": "ConTotal",
    "risk": "medium",
    "riskLabel": "Riesgo Medio",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 2,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ProjectSQL",
    "name": "ProjectSQL",
    "javaFiles": 9,
    "testFiles": 0,
    "hasPom": true,
    "score": 33.3,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 3,
      "sec": 0,
      "xml": 6
    },
    "initialList": "initial"
  },
  {
    "id": "NormativosUtilidades",
    "name": "NormativosUtilidades",
    "javaFiles": 10,
    "testFiles": 0,
    "hasPom": false,
    "score": 33,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargadorSCIs",
    "name": "RDR_CargadorSCIs",
    "javaFiles": 10,
    "testFiles": 0,
    "hasPom": false,
    "score": 33,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "UtilsCSV",
    "name": "UtilsCSV",
    "javaFiles": 10,
    "testFiles": 0,
    "hasPom": false,
    "score": 33,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "AlertFXFndmngrReceive",
    "name": "AlertFXFndmngrReceive",
    "javaFiles": 9,
    "testFiles": 0,
    "hasPom": false,
    "score": 32.7,
    "process": "ALERT_FX_REPORT",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 3
    },
    "initialList": "initial"
  },
  {
    "id": "clientelaBDI_Altas_response",
    "name": "clientelaBDI_Altas_response",
    "javaFiles": 9,
    "testFiles": 0,
    "hasPom": false,
    "score": 32.1,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_ConciliaInstLiqAbaco5",
    "name": "RDR_ConciliaInstLiqAbaco5",
    "javaFiles": 9,
    "testFiles": 0,
    "hasPom": false,
    "score": 32.1,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "AltasBajas_Indices",
    "name": "AltasBajas_Indices",
    "javaFiles": 9,
    "testFiles": 0,
    "hasPom": false,
    "score": 32.1,
    "process": "AltaFondos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "clientelaBDI_Altas_responseSCF",
    "name": "clientelaBDI_Altas_responseSCF",
    "javaFiles": 9,
    "testFiles": 0,
    "hasPom": false,
    "score": 32.1,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "LEI_Register_response",
    "name": "LEI_Register_response",
    "javaFiles": 9,
    "testFiles": 0,
    "hasPom": false,
    "score": 32.1,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargadorSDIs",
    "name": "RDR_CargadorSDIs",
    "javaFiles": 8,
    "testFiles": 0,
    "hasPom": false,
    "score": 31.2,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ExtraccionGenericaCPTY",
    "name": "ExtraccionGenericaCPTY",
    "javaFiles": 8,
    "testFiles": 0,
    "hasPom": false,
    "score": 31.2,
    "process": "Extraccion-Generica",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_ConciliaSwiftAbaco",
    "name": "RDR_ConciliaSwiftAbaco",
    "javaFiles": 8,
    "testFiles": 0,
    "hasPom": false,
    "score": 31.2,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_ConciliaInstLiqAbaco",
    "name": "RDR_ConciliaInstLiqAbaco",
    "javaFiles": 8,
    "testFiles": 0,
    "hasPom": false,
    "score": 31.2,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ReporteSimulacionCierreOficinas",
    "name": "ReporteSimulacionCierreOficinas",
    "javaFiles": 8,
    "testFiles": 0,
    "hasPom": false,
    "score": 31.2,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargadorContactos",
    "name": "RDR_CargadorContactos",
    "javaFiles": 8,
    "testFiles": 0,
    "hasPom": false,
    "score": 31.2,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "Investors_Client_Reg_resp",
    "name": "Investors_Client_Reg_resp",
    "javaFiles": 8,
    "testFiles": 0,
    "hasPom": false,
    "score": 31.2,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "DataQualityContact",
    "name": "DataQualityContact",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargaCAC1",
    "name": "RDR_CargaCAC1",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RC_Contacts",
    "name": "RC_Contacts",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "clientelaBDI_Altas_request",
    "name": "clientelaBDI_Altas_request",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "maestroEmisionesRDR2",
    "name": "maestroEmisionesRDR2",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Indices",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_GestionCpartyC460",
    "name": "RDR_GestionCpartyC460",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "LEI_Register_request",
    "name": "LEI_Register_request",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_PLSQL",
    "name": "RDR_PLSQL",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "maestroEmisionesRDR",
    "name": "maestroEmisionesRDR",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Indices",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_cuentaBajaAlta",
    "name": "RDR_cuentaBajaAlta",
    "javaFiles": 7,
    "testFiles": 0,
    "hasPom": false,
    "score": 30.3,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargaCacheOLAP",
    "name": "RDR_CargaCacheOLAP",
    "javaFiles": 4,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.8,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 11
    },
    "initialList": "initial"
  },
  {
    "id": "TraductorXML",
    "name": "TraductorXML",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.5,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 14
    },
    "initialList": "initial"
  },
  {
    "id": "AltamiraMexicoConciliacion",
    "name": "AltamiraMexicoConciliacion",
    "javaFiles": 6,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.4,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "LegalOpinionResponse",
    "name": "LegalOpinionResponse",
    "javaFiles": 6,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.4,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_AlertasCocinado",
    "name": "RDR_AlertasCocinado",
    "javaFiles": 6,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.4,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "AltaFondos_CuadreCarga",
    "name": "AltaFondos_CuadreCarga",
    "javaFiles": 6,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.4,
    "process": "AltaFondos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "CSVToXML_Layout",
    "name": "CSVToXML_Layout",
    "javaFiles": 6,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.4,
    "process": "AltaFondos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_ConciliaContactosAbaco",
    "name": "RDR_ConciliaContactosAbaco",
    "javaFiles": 6,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.4,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "Refinitiv_Ratings",
    "name": "Refinitiv_Ratings",
    "javaFiles": 6,
    "testFiles": 0,
    "hasPom": false,
    "score": 29.4,
    "process": "Refinitiv-Loader",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CrearIndices_Emisiones",
    "name": "RDR_CrearIndices_Emisiones",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Indices",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_INFCIERRE_REPORTE",
    "name": "RDR_INFCIERRE_REPORTE",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Historificacion_Emisiones",
    "name": "RDR_Historificacion_Emisiones",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Indices",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargaCacheCAC1",
    "name": "RDR_CargaCacheCAC1",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "maestroEmisionesMergeRDR",
    "name": "maestroEmisionesMergeRDR",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Indices",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "SDIs_Duplicidades",
    "name": "SDIs_Duplicidades",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "SimulacionCierreOficinas",
    "name": "SimulacionCierreOficinas",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ReporteInfCierre",
    "name": "ReporteInfCierre",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_AlertasBarrido",
    "name": "RDR_AlertasBarrido",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Report",
    "name": "RDR_Report",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": false,
    "score": 28.5,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDRCommon",
    "name": "RDRCommon",
    "javaFiles": 11,
    "testFiles": 1,
    "hasPom": false,
    "score": 28.3,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 2
    },
    "initialList": "initial"
  },
  {
    "id": "API_REST",
    "name": "API_REST",
    "javaFiles": 4,
    "testFiles": 0,
    "hasPom": false,
    "score": 27.6,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "SCIs_Duplicidades",
    "name": "SCIs_Duplicidades",
    "javaFiles": 4,
    "testFiles": 0,
    "hasPom": false,
    "score": 27.6,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "Lector_IdRDR",
    "name": "Lector_IdRDR",
    "javaFiles": 4,
    "testFiles": 0,
    "hasPom": false,
    "score": 27.6,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ReporteOFS",
    "name": "ReporteOFS",
    "javaFiles": 4,
    "testFiles": 0,
    "hasPom": false,
    "score": 27.6,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Load_RGA",
    "name": "RDR_Load_RGA",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 27.2,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 7
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CSVToXML",
    "name": "RDR_CSVToXML",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.9,
    "process": "AltaFondos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 1
    },
    "initialList": "initial"
  },
  {
    "id": "ExtraccionGenericaEMISI",
    "name": "ExtraccionGenericaEMISI",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "Extraccion-Generica",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_ME_Load_Warrants",
    "name": "RDR_ME_Load_Warrants",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "MapeoSIS",
    "name": "MapeoSIS",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "CargaRatingsInternos",
    "name": "CargaRatingsInternos",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "Refinitiv-Loader",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ControlOnBoarding",
    "name": "ControlOnBoarding",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ControlCargaDatos",
    "name": "ControlCargaDatos",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "ConTotal",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_MapeoSSIs",
    "name": "RDR_MapeoSSIs",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ProcesoFusion",
    "name": "ProcesoFusion",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "CtpdaModifPlaza",
    "name": "CtpdaModifPlaza",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.7,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "Transformar_XML",
    "name": "Transformar_XML",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.3,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 7
    },
    "initialList": "initial"
  },
  {
    "id": "ProductosBatch",
    "name": "ProductosBatch",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.3,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 7
    },
    "initialList": "initial"
  },
  {
    "id": "EmisionesValidation",
    "name": "EmisionesValidation",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 26.1,
    "process": "Indices",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 6
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_GenericValidatorXSD",
    "name": "RDR_GenericValidatorXSD",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 5
    },
    "initialList": "initial"
  },
  {
    "id": "mentor",
    "name": "mentor",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "informeBroker",
    "name": "informeBroker",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ValuationForResolution",
    "name": "ValuationForResolution",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "DatetoSSIS",
    "name": "DatetoSSIS",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_OFS_REPORTE",
    "name": "RDR_OFS_REPORTE",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Warrants_HAB",
    "name": "RDR_Warrants_HAB",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Extraccion_RGA",
    "name": "RDR_Extraccion_RGA",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_InformeBroker",
    "name": "RDR_InformeBroker",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "nfc",
    "name": "nfc",
    "javaFiles": 2,
    "testFiles": 0,
    "hasPom": false,
    "score": 25.8,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_salesWarehouse",
    "name": "RDR_salesWarehouse",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_SW",
    "name": "RDR_SW",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ReplaceStringtoEmptyProject",
    "name": "ReplaceStringtoEmptyProject",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ReplaceString",
    "name": "ReplaceString",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "informeMIFID",
    "name": "informeMIFID",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Mifid-II",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "HistAndZip",
    "name": "HistAndZip",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_DeltaEmisores",
    "name": "RDR_DeltaEmisores",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "Morningcheck",
    "name": "Morningcheck",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "rdrmaestroemisiones",
    "name": "rdrmaestroemisiones",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Indices",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "xslx2csv",
    "name": "xslx2csv",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_InformeMIFID",
    "name": "RDR_InformeMIFID",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Mifid-II",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ConexionBD",
    "name": "ConexionBD",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "ConTotal",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDRconciliacion",
    "name": "RDRconciliacion",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_DELTA_EMISORES",
    "name": "RDR_DELTA_EMISORES",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Envio_Reportes",
    "name": "RDR_Envio_Reportes",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "DecomisadoNRO",
    "name": "DecomisadoNRO",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": false,
    "score": 24.9,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ProjectMain",
    "name": "ProjectMain",
    "javaFiles": 15,
    "testFiles": 0,
    "hasPom": true,
    "score": 20.7,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 6
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_ReportMail",
    "name": "RDR_ReportMail",
    "javaFiles": 18,
    "testFiles": 7,
    "hasPom": true,
    "score": 16.2,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_LoadBasketsAdHoc",
    "name": "RDR_LoadBasketsAdHoc",
    "javaFiles": 11,
    "testFiles": 0,
    "hasPom": true,
    "score": 15.9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "openFigi",
    "name": "openFigi",
    "javaFiles": 9,
    "testFiles": 0,
    "hasPom": true,
    "score": 14.1,
    "process": "Extraccion-Generica",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ExtraccionGenericaOtherEntities",
    "name": "ExtraccionGenericaOtherEntities",
    "javaFiles": 15,
    "testFiles": 6,
    "hasPom": true,
    "score": 13.5,
    "process": "Extraccion-Generica",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "MENTOR_Difusion",
    "name": "MENTOR_Difusion",
    "javaFiles": 13,
    "testFiles": 6,
    "hasPom": true,
    "score": 11.7,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "XMASToken",
    "name": "XMASToken",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": true,
    "score": 11.1,
    "process": "Extraccion-Generica",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 3
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargadorIssueMex",
    "name": "RDR_CargadorIssueMex",
    "javaFiles": 5,
    "testFiles": 0,
    "hasPom": true,
    "score": 10.5,
    "process": "ISSUMEX",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ExtraccionGenericaUnificada",
    "name": "ExtraccionGenericaUnificada",
    "javaFiles": 4,
    "testFiles": 0,
    "hasPom": true,
    "score": 10,
    "process": "Extraccion-Generica",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 2
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargadorThirdparties",
    "name": "RDR_CargadorThirdparties",
    "javaFiles": 10,
    "testFiles": 5,
    "hasPom": true,
    "score": 9,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "rdr_bajas_comex",
    "name": "rdr_bajas_comex",
    "javaFiles": 3,
    "testFiles": 0,
    "hasPom": true,
    "score": 8.7,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_FormatoUnicoBaskets",
    "name": "RDR_FormatoUnicoBaskets",
    "javaFiles": 1,
    "testFiles": 0,
    "hasPom": true,
    "score": 8.5,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 8
    },
    "initialList": "initial"
  },
  {
    "id": "Investors_csv_PreProcess",
    "name": "Investors_csv_PreProcess",
    "javaFiles": 5,
    "testFiles": 5,
    "hasPom": true,
    "score": 4.5,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_CargadorSubaccounts",
    "name": "RDR_CargadorSubaccounts",
    "javaFiles": 5,
    "testFiles": 5,
    "hasPom": true,
    "score": 4.5,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "ProjectMain_Aux",
    "name": "ProjectMain_Aux",
    "javaFiles": 0,
    "testFiles": 0,
    "hasPom": false,
    "score": 24,
    "process": "Otros Proyectos",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  },
  {
    "id": "RDR_Testing_Tool",
    "name": "RDR_Testing_Tool",
    "javaFiles": 0,
    "testFiles": 0,
    "hasPom": false,
    "score": 24,
    "process": "Services-RDR",
    "risk": "low",
    "riskLabel": "Riesgo Bajo",
    "comment": "",
    "tech": {
      "jaxb": 0,
      "jpa": 0,
      "jms": 0,
      "sun": 0,
      "sec": 0,
      "xml": 0
    },
    "initialList": "initial"
  }
];

    let projects = [];

    // Exact Columns configuration from the template
    const COLUMNS = [
      { id: 'initial', label: 'Analisis Inicial', icon: '📋' },
      { id: 'maven', label: 'Mavenizacion', icon: '🏗️' },
      { id: 'tests-java8', label: 'Tests Java 8', icon: '✅' },
      { id: 'java17-upgrade', label: 'Upgrade JDK 17', icon: '🔄' },
      { id: 'tests-java17', label: 'Tests JDK 17', icon: '🧪' },
      { id: 'completed', label: 'Completado', icon: '✨' }
    ];

    // Processes (Swimlanes) configuration
    const PROCESSES = [
      { id: 'ALERT_FX_REPORT', name: 'ALERT_FX_REPORT' },
      { id: 'AltaFondos', name: 'AltaFondos' },
      { id: 'Calypso-Filters', name: 'Calypso-Filters' },
      { id: 'ConTotal', name: 'ConTotal' },
      { id: 'Extraccion-Generica', name: 'Extraccion-Generica' },
      { id: 'Indices', name: 'Indices' },
      { id: 'ISSUMEX', name: 'ISSUMEX' },
      { id: 'Mifid-II', name: 'Mifid-II' },
      { id: 'Refinitiv-Loader', name: 'Refinitiv-Loader' },
      { id: 'Rules-Engine', name: 'Rules-Engine' },
      { id: 'Services-RDR', name: 'Services-RDR' },
      { id: 'Otros Proyectos', name: 'Otros Proyectos' }
    ];

    // Set visual syncing indicator
    function setSyncState(state, message) {
      const dot = document.getElementById('syncDot');
      const text = document.getElementById('syncText');
      dot.className = 'status-dot ' + state;
      text.textContent = message;
    }

    // Show Toast
    function showToast(message) {
      const toast = document.createElement('div');
      toast.className = 'toast';
      toast.textContent = message;
      document.body.appendChild(toast);
      setTimeout(() => toast.remove(), 4000);
    }

    // Load state from Google Sheets
    async function loadData() {
      if (!SCRIPT_URL || SCRIPT_URL.includes("TU_GOOGLE_APPS_SCRIPT")) {
        console.warn("Google Sheets API URL not configured. Falling back to local state.");
        document.getElementById('loadingOverlay').style.display = 'none';
        setSyncState('disconnected', 'Modo Local');
        initLocalData();
        renderBoard();
        return;
      }

      setSyncState('syncing', 'Sincronizando...');
      document.getElementById('loadingOverlay').style.display = 'flex';
      document.getElementById('loadingText').innerHTML = '<div class="spinner"></div><div>Descargando estado desde Google Sheets...</div>';

      try {
        const response = await fetch(SCRIPT_URL + "?action=read");
        if (!response.ok) throw new Error("HTTP error " + response.status);
        
        const sheetData = await response.json();
        
        projects = JSON.parse(JSON.stringify(initialProjects));
        
        const sheetMap = {};
        sheetData.forEach(row => {
          sheetMap[row.projectId] = row;
        });

        projects.forEach(p => {
          const row = sheetMap[p.id];
          if (row) {
            p.currentList = row.listId;
            if (row.checklist && p.checklist) {
              const states = row.checklist.split(',');
              p.checklist.forEach((item, idx) => {
                if (states[idx] !== undefined) {
                  item.done = states[idx] === '1';
                }
              });
            }
          } else {
            p.currentList = p.initialList;
          }
        });

        setSyncState('connected', 'Sincronizado');
        document.getElementById('loadingOverlay').style.display = 'none';
        showToast('🔄 Datos actualizados de Google Sheets');
        renderBoard();

      } catch (err) {
        console.error("Failed to fetch from Google Sheets:", err);
        setSyncState('disconnected', 'Usando Local (Sin Conexión)');
        document.getElementById('loadingText').innerHTML = 
          "<span style='color:var(--bad); font-weight:700;'>Error de conexión con Google Sheets</span><br>" +
          "<span style='font-size:12px; color:var(--muted);'>" + err.message + "</span><br><br>" +
          "<button onclick='bypassToLocal()' class='btn btn-refresh' style='background:var(--accent);color:white;'>Continuar sin conexión</button>";
      }
    }

    function bypassToLocal() {
      document.getElementById('loadingOverlay').style.display = 'none';
      initLocalData();
      renderBoard();
    }

    function initLocalData() {
      const savedState = localStorage.getItem('exact_board_local_fallback');
      if (savedState) {
        try {
          projects = JSON.parse(savedState);
        } catch(e) {
          projects = JSON.parse(JSON.stringify(initialProjects));
        }
      } else {
        projects = JSON.parse(JSON.stringify(initialProjects));
      }
    }

    function saveLocalState() {
      localStorage.setItem('exact_board_local_fallback', JSON.stringify(projects));
    }

    // Sync card updates to Sheets
    async function syncCard(p) {
      if (!SCRIPT_URL || SCRIPT_URL.includes("TU_GOOGLE_APPS_SCRIPT")) {
        saveLocalState();
        return;
      }

      setSyncState('syncing', 'Guardando...');
      
      const checklistStr = p.checklist ? p.checklist.map(item => item.done ? '1' : '0').join(',') : '';
      const updateUrl = SCRIPT_URL + 
        "?action=update" +
        "&projectId=" + encodeURIComponent(p.id) +
        "&listId=" + encodeURIComponent(p.currentList) +
        "&checklist=" + encodeURIComponent(checklistStr);

      try {
        const response = await fetch(updateUrl, { method: 'GET', keepalive: true });
        if (!response.ok) throw new Error("HTTP error " + response.status);
        setSyncState('connected', 'Sincronizado');
        updateSwimlaneProgress(p.process);
      } catch (err) {
        console.error("Failed to sync card:", err);
        setSyncState('disconnected', 'Error de conexión');
      }
    }

    // Drag and Drop variables
    let draggedCardId = null;

    function drag(event) {
      draggedCardId = event.target.dataset.id;
      event.target.classList.add('dragging');
    }

    function dragEnd(event) {
      event.target.classList.remove('dragging');
    }

    function allowDrop(event) {
      event.preventDefault();
    }

    function drop(event, processId, targetListId) {
      event.preventDefault();
      if (!draggedCardId) return;

      const project = projects.find(p => p.id === draggedCardId);
      // Cards can only be dragged within their own process (swimlane)
      if (project && project.process === processId && project.currentList !== targetListId) {
        project.currentList = targetListId;
        renderBoard();
        syncCard(project);
      }
      draggedCardId = null;
    }

    function toggleChecklistItem(projectId, index) {
      const p = projects.find(proj => proj.id === projectId);
      if (p && p.checklist) {
        const checkbox = document.getElementById("chk-" + p.id + "-" + index);
        p.checklist[index].done = checkbox.checked;
        saveLocalState();
        syncCard(p);
      }
    }

    function updateSwimlaneProgress(processId) {
      const blockProjects = projects.filter(p => p.process === processId);
      let totalTasks = 0;
      let completedTasks = 0;

      blockProjects.forEach(p => {
        if (p.checklist) {
          totalTasks += p.checklist.length;
          completedTasks += p.checklist.filter(item => item.done).length;
        }
      });

      const percentage = totalTasks > 0 ? Math.round((completedTasks / totalTasks) * 100) : 0;
      
      const bar = document.getElementById("progress-bar-" + processId);
      const text = document.getElementById("progress-text-" + processId);
      if (bar && text) {
        bar.style.width = percentage + "%";
        text.textContent = percentage + "% completados";
      }
    }

    function filterSwimlanes() {
      const search = document.getElementById('searchFilter').value.toLowerCase();
      const risk = document.getElementById('filterRisk').value;
      
      const swimlanes = document.querySelectorAll('.swimlane');
      swimlanes.forEach(sl => {
        const processId = sl.dataset.process;
        
        // Find if any card in this swimlane matches the filters
        const cards = sl.querySelectorAll('.card');
        let hasMatches = false;
        
        cards.forEach(card => {
          const name = card.dataset.name.toLowerCase();
          const cardRisk = card.dataset.risk;
          const matchesSearch = name.includes(search);
          const matchesRisk = risk === 'all' || cardRisk === risk;
          
          if (matchesSearch && matchesRisk) {
            card.style.display = 'flex';
            hasMatches = true;
          } else {
            card.style.display = 'none';
          }
        });

        // Hide columns description empty divs if we are filtering
        const emptyDivs = sl.querySelectorAll('.column-empty');
        emptyDivs.forEach(ed => {
          ed.style.display = (search !== '' || risk !== 'all') ? 'none' : 'block';
        });

        sl.style.display = (hasMatches || (search === '' && risk === 'all')) ? 'block' : 'none';
      });
    }

    // Render Board
    function renderBoard() {
      const container = document.getElementById('swimlanes-container');
      container.innerHTML = '';

      // Initialize missing project categories
      projects.forEach(p => {
        if (!p.currentList) {
          p.currentList = p.initialList;
        }
        if (!p.checklist) {
          p.checklist = [
            { text: "Tests detectados", done: p.testFiles > 0 },
            { text: "Maven configurado", done: p.hasPom },
            { text: "Tests Java 8 ✓", done: false },
            { text: "JDK 17 actualizado", done: false },
            { text: "Tests JDK 17 ✓", done: false }
          ];
        }
      });

      // Render Swimlanes
      PROCESSES.forEach(proc => {
        const processProjects = projects.filter(p => p.process === proc.id);
        
        if (processProjects.length === 0) return; // Don't render empty swimlanes

        const swimlane = document.createElement('div');
        swimlane.className = 'swimlane';
        swimlane.dataset.process = proc.id;

        // Header
        const header = document.createElement('div');
        header.className = 'swimlane-header';

        const titleInfo = document.createElement('div');
        titleInfo.style.flex = '1';

        const titleText = document.createElement('h3');
        titleText.className = 'swimlane-title';
        titleText.textContent = proc.name;
        titleInfo.appendChild(titleText);

        const info = document.createElement('div');
        info.className = 'swimlane-info';

        const totalEffort = processProjects.reduce((sum, p) => sum + p.score, 0);
        const completedCount = processProjects.filter(p => p.currentList === 'completed').length;
        const progress = processProjects.length > 0 ? (completedCount / processProjects.length) * 100 : 0;

        info.innerHTML = 
          '<span id="progress-text-' + proc.id + '">' + completedCount + '/' + processProjects.length + ' completados</span>' +
          '<div class="progress-indicator">' +
            '<div class="progress-bar" id="progress-bar-' + proc.id + '"></div>' +
          '</div>' +
          '<span>Complejidad: ' + totalEffort.toFixed(1) + '</span>';
        
        titleInfo.appendChild(info);
        header.appendChild(titleInfo);
        swimlane.appendChild(header);

        // Columns Wrapper
        const columnsWrapper = document.createElement('div');
        columnsWrapper.className = 'columns-wrapper';

        COLUMNS.forEach(column => {
          const col = document.createElement('div');
          col.className = 'column';
          col.dataset.column = column.id;
          col.dataset.process = proc.id;

          const colHeader = document.createElement('div');
          colHeader.className = 'column-header';
          
          const colProjects = processProjects.filter(p => p.currentList === column.id);

          colHeader.innerHTML = 
            '<span><span class="column-icon">' + column.icon + '</span>' + column.label + '</span>' +
            '<span class="column-count">' + colProjects.length + '</span>';
          col.appendChild(colHeader);

          const cardsContainer = document.createElement('div');
          cardsContainer.className = 'cards-container';
          cardsContainer.ondragover = allowDrop;
          cardsContainer.ondrop = (e) => drop(e, proc.id, column.id);

          if (colProjects.length === 0) {
            const empty = document.createElement('div');
            empty.className = 'column-empty';
            empty.textContent = '−';
            cardsContainer.appendChild(empty);
          } else {
            colProjects.forEach(p => {
              const card = document.createElement('div');
              card.className = 'card';
              card.draggable = true;
              card.dataset.id = p.id;
              card.dataset.name = p.name;
              card.dataset.risk = p.risk;
              card.dataset.column = column.id;

              const cardHeader = document.createElement('div');
              cardHeader.className = 'card-header';
              cardHeader.textContent = p.name;

              const risk = document.createElement('div');
              risk.className = 'card-risk risk-' + p.risk;
              risk.textContent = p.riskLabel.toUpperCase();

              const cardMeta = document.createElement('div');
              cardMeta.className = 'card-meta';
              cardMeta.innerHTML = 'Complejidad: ' + p.score.toFixed(1) + ' • ' + (p.hasPom ? '✓ Maven' : '✗ Sin Maven');

              const checklist = document.createElement('ul');
              checklist.className = 'checklist';

              p.checklist.forEach((item, idx) => {
                const checked = item.done ? 'checked' : '';
                const checklistItem = document.createElement('li');
                checklistItem.className = 'checklist-item';

                checklistItem.innerHTML = 
                  '<input type="checkbox" id="chk-' + p.id + '-' + idx + '" ' + checked + ' onchange="toggleChecklistItem(\'' + p.id + '\', ' + idx + ')">' +
                  '<label for="chk-' + p.id + '-' + idx + '">' + item.text + '</label>';
                
                checklist.appendChild(checklistItem);
              });

              card.appendChild(cardHeader);
              card.appendChild(risk);
              card.appendChild(cardMeta);
              card.appendChild(checklist);

              // Drag & drop listeners
              card.ondragstart = drag;
              card.ondragend = dragEnd;

              cardsContainer.appendChild(card);
            });
          }

          col.appendChild(cardsContainer);
          columnsWrapper.appendChild(col);
        });

        swimlane.appendChild(columnsWrapper);
        container.appendChild(swimlane);

        // Update progress bar
        updateSwimlaneProgress(proc.id);
      });
    }

    // Set event listeners
    document.getElementById('btnRefresh').addEventListener('click', loadData);

    // Drag over document behavior
    document.addEventListener('dragover', (e) => {
      e.preventDefault();
      e.dataTransfer.dropEffect = 'move';
    });

    document.addEventListener('drop', (e) => {
      e.preventDefault();
      const column = e.target.closest('.column');
      if (column) {
        const java = e.dataTransfer.getData('java') || draggedCardId;
        const fromColumn = e.dataTransfer.getData('fromColumn');
        const toColumn = column.getAttribute('data-column');
        const processId = column.getAttribute('data-process');
        
        const project = projects.find(p => p.id === java);
        if (project && project.process === processId && toColumn && project.currentList !== toColumn) {
          project.currentList = toColumn;
          renderBoard();
          syncCard(project);
        }
      }
      draggedCardId = null;
    });

    // Start
    loadData();
  </script>
</body>
</html>
