# sas_wizz
Sas html viz. scripts, mostly gemini, to start out


/* ================================================================================= */
/* STEP 1: CREATE MOCK DATA TABLES                                                   */
/* ================================================================================= */

/* Table A: Department Stats */
data work.dep_stats;
    input DepName $ DepID $ No_Cases Avg_Satisfaction;
    datalines;
Sales D01 150 4.5
IT    D02 85  3.2
HR    D03 40  4.8
Marketing D04 95 4.1
;
run;

/* Table B: Employee Stats */
data work.emp_stats;
    input EmpName $ EmpID $ DepID $ No_Cases Total_Hours;
    datalines;
Adam  E01 D01 10 120
Eve   E02 D01 15 160
Steve E03 D02 8  90
Sarah E04 D02 12 110
John  E05 D03 5  40
Mike  E06 D04 22 180
;
run;

/* Table C: CORE DATA (Transactions) */
data work.core_data;
    length CaseType $12;
    input EmpID $ EmpName $ CaseID $ CaseType $ DepID $ DepName $ Indiv_Hours Full_Case_Hours;
    datalines;
E01 Adam C101 Deal_Close D01 Sales 10 20
E01 Adam C102 Meeting    D01 Sales 5  5
E01 Adam C103 Paperwork  D01 Sales 8  8
E02 Eve  C101 Deal_Close D01 Sales 10 20
E02 Eve  C104 Prospect   D01 Sales 20 20
E03 Steve C201 Bug_Fix   D02 IT    15 15
E03 Steve C202 Server    D02 IT    4  8
E04 Sarah C202 Server    D02 IT    4  8
E04 Sarah C203 Security  D02 IT    10 10
E05 John  C301 Hiring    D03 HR    8  8
E06 Mike  C401 Campaign  D04 Marketing 12 24
E06 Mike  C402 Ad_Buy    D04 Marketing 5 5
;
run;

/* Prepare Dropdown Lists */
proc sql noprint;
    select distinct DepName into :dept_list separated by '|' from work.dep_stats;
    select distinct EmpName into :emp_list separated by '|' from work.emp_stats;
quit;




# Dashboard generator

/* ================================================================================= */
/* CONFIGURATION                                                                     */
/* ================================================================================= */
%let outpath = C:\temp;
%let filename = Seaborn_Dashboard.html;

ods _all_ close;

/* ================================================================================= */
/* GENERATE HTML REPORT                                                              */
/* ================================================================================= */
ods html file="&outpath\&filename" path="&outpath"
    /* We use 'htmlblue' as a base, but override everything with CSS below */
    style=styles.htmlblue
    headtext='
    <link rel="stylesheet" type="text/css" href="https://cdn.datatables.net/1.10.20/css/jquery.dataTables.css">
    <script src="https://code.jquery.com/jquery-3.3.1.js"></script>
    <script src="https://cdn.datatables.net/1.10.20/js/jquery.dataTables.js"></script>
    
    <style>
        /* --- 1. SEABORN PALETTE & GLOBAL THEME --- */
        :root {
            --sea-blue: #4C72B0;   /* Muted Blue */
            --sea-green: #55A868;  /* Muted Green */
            --sea-red: #C44E52;    /* Muted Red */
            --bg-color: #EAEAF2;   /* Seaborn Grid Grey */
            --text-color: #333333;
        }

        body { 
            font-family: "Segoe UI", Roboto, Helvetica, Arial, sans-serif; 
            background-color: var(--bg-color); 
            margin: 0; padding: 0; 
            color: var(--text-color);
        }

        /* --- 2. HEADER BAR --- */
        .header {
            background-color: white;
            padding: 20px 40px;
            border-bottom: 1px solid #ddd;
            display: flex; justify-content: space-between; align-items: center;
        }
        h1 { margin: 0; font-size: 24px; color: var(--sea-blue); font-weight: 600; }
        
        /* --- 3. NAVIGATION TABS (Clean White Style) --- */
        .tab-nav { 
            background-color: white; 
            padding: 0 40px; 
            box-shadow: 0 2px 4px rgba(0,0,0,0.05);
        }
        .tab-nav button { 
            background: transparent; 
            border: none; 
            border-bottom: 3px solid transparent; /* Hidden underline */
            padding: 15px 25px; 
            font-size: 15px; 
            cursor: pointer; 
            color: #666;
            font-weight: 500;
            transition: 0.3s;
        }
        .tab-nav button:hover { color: var(--sea-blue); background: #f9f9f9; }
        
        /* The Active Tab gets the Blue Underline */
        .tab-nav button.active { 
            color: var(--sea-blue); 
            border-bottom: 3px solid var(--sea-blue); 
        }

        /* --- 4. CONTENT CARDS --- */
        .main-container { padding: 30px 40px; }
        
        .tab-content { 
            display: none; 
            animation: fadeIn 0.3s ease-in-out;
        }
        
        /* The "Card" Look for sections */
        .card {
            background: white;
            padding: 25px;
            border-radius: 4px;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
            margin-bottom: 25px;
        }

        /* --- 5. CONTROLS --- */
        .control-panel { 
            background: #f4f6f9; 
            padding: 15px; 
            border-radius: 4px; 
            border-left: 5px solid var(--sea-green);
            margin-bottom: 20px;
        }
        select { 
            padding: 8px 12px; border: 1px solid #ccc; border-radius: 4px; font-size: 14px; 
        }
        
        /* Helper Classes */
        .graph-container { display: none; }
        h3 { margin-top: 0; color: #444; font-size: 18px; border-bottom: 1px solid #eee; padding-bottom: 10px; }
        
        @keyframes fadeIn { from { opacity: 0; transform: translateY(5px); } to { opacity: 1; transform: translateY(0); } }
    </style>

    <script>
        function openTab(evt, tabName) {
            $(".tab-content").hide();
            $(".tablinks").removeClass("active");
            $("#" + tabName).show();
            $(evt.currentTarget).addClass("active");
        }
        function filterDept(val) { $(".dept-drill").hide(); $("#D_"+val).fadeIn(); }
        function filterEmp(val) { $(".emp-drill").hide(); $("#E_"+val).fadeIn(); }
        
        $(document).ready(function() {
            $(".search-table").DataTable();
            document.getElementById("defaultOpen").click();
        });
    </script>
    ';

/* --- PAGE STRUCTURE --- */
ods html text='<div class="header"><h1>Strategic Dashboard</h1><p style="color:#888; margin:0;">Q1 Performance Report</p></div>';

/* --- NAVIGATION --- */
ods html text='
<div class="tab-nav">
  <button class="tablinks" onclick="openTab(event, ''TabDept'')" id="defaultOpen">Department Stats</button>
  <button class="tablinks" onclick="openTab(event, ''TabEmp'')">Employee Stats</button>
  <button class="tablinks" onclick="openTab(event, ''TabData'')">Data Vault</button>
</div>
<div class="main-container"> ';

/* ============================================================ */
/* TAB 1: DEPARTMENT ANALYSIS                                   */
/* ============================================================ */
ods html text='<div id="TabDept" class="tab-content">';
    
    /* CARD 1: OVERVIEW */
    ods html text='<div class="card">';
    ods html text='<h3>Departmental Volume</h3>';
    
        /* NOTE: styleattrs datacolors=() forces the colors directly */
        proc sgplot data=work.dep_stats;
            vbar DepName / response=No_Cases group=DepName;
            styleattrs datacolors=(cx4C72B0 cx55A868 cxC44E52 cx8172B2); /* Muted Blue, Green, Red, Purple */
            yaxis grid; 
        run;
    ods html text='</div>';

    /* CARD 2: DRILL DOWN */
    ods html text='<div class="card">';
    ods html text='<h3>Case Breakdown</h3>';
    ods html text='<div class="control-panel"><b>Filter by Department: </b><select onchange="filterDept(this.value)"><option>-- Select --</option>';
    %macro dept_drop;
        %let count = %sysfunc(countw(&dept_list, |));
        %do i = 1 %to &count;
            %let d = %scan(&dept_list, &i, |);
            ods html text="<option value='&d'>&d</option>";
        %end;
    %mend;
    %dept_drop;
    ods html text='</select></div>';

    /* Drill-Down Graphs */
    %macro dept_drill;
        %let count = %sysfunc(countw(&dept_list, |));
        %do i = 1 %to &count;
            %let d = %scan(&dept_list, &i, |);
            ods html text="<div id='D_&d' class='dept-drill graph-container'>";
                
                proc sgplot data=work.core_data;
                    where DepName = "&d";
                    /* FORCED COLOR: Muted Blue with transparency */
                    hbar CaseType / response=Indiv_Hours fillattrs=(color=cx4C72B0 transparency=0.1); 
                    xaxis grid;
                    title "Hours by Case Type: &d";
                run;
                
            ods html text="</div>";
        %end;
    %mend;
    %dept_drill;
    ods html text='</div>'; /* End Card */

ods html text='</div>'; /* End Tab */

/* ============================================================ */
/* TAB 2: EMPLOYEE ANALYSIS                                     */
/* ============================================================ */
ods html text='<div id="TabEmp" class="tab-content">';

    /* CARD 1: OVERVIEW */
    ods html text='<div class="card">';
    ods html text='<h3>Top Performers</h3>';
    proc sgplot data=work.emp_stats;
        vbar EmpName / response=No_Cases categoryorder=respdesc fillattrs=(color=cx55A868); /* Muted Green */
        yaxis grid;
    run;
    ods html text='</div>';

    /* CARD 2: DRILL DOWN */
    ods html text='<div class="card">';
    ods html text='<h3>Workload Comparison</h3>';
    ods html text='<div class="control-panel"><b>Select Employee: </b><select onchange="filterEmp(this.value)"><option>-- Select --</option>';
    %macro emp_drop;
        %let count = %sysfunc(countw(&emp_list, |));
        %do i = 1 %to &count;
            %let e = %scan(&emp_list, &i, |);
            ods html text="<option value='&e'>&e</option>";
        %end;
    %mend;
    %emp_drop;
    ods html text='</select></div>';

    /* Drill-Down Graphs */
    %macro emp_drill;
        %let count = %sysfunc(countw(&emp_list, |));
        %do i = 1 %to &count;
            %let e = %scan(&emp_list, &i, |);
            ods html text="<div id='E_&e' class='emp-drill graph-container'>";
                
                proc sgplot data=work.core_data;
                    where EmpName = "&e";
                    /* Compare Blue (Mine) vs Grey (Total) */
                    vbar CaseID / response=Indiv_Hours name="Mine" legendlabel="User Hours" fillattrs=(color=cx4C72B0);
                    vbar CaseID / response=Full_Case_Hours name="Full" legendlabel="Total Case Hours" fillattrs=(color=cx777777 transparency=0.7);
                    keylegend "Mine" "Full" / noborder location=inside position=topright;
                    yaxis grid;
                    title "Efficiency Check: &e";
                run;
                
            ods html text="</div>";
        %end;
    %mend;
    %emp_drill;
    ods html text='</div>'; /* End Card */

ods html text='</div>';

/* ============================================================ */
/* TAB 3: DATA VAULT                                            */
/* ============================================================ */
ods html text='<div id="TabData" class="tab-content">';
    ods html text='<div class="card">';
    ods html text='<h3>Master Database</h3>';
    
    proc print data=work.core_data noobs label 
        style(table)={htmlclass="search-table" width="100%" frame=box rules=rows background=white bordercolor=cxEAEAF2};
    run;
    
    ods html text='</div>';
ods html text='</div>';

ods html text='</div>'; /* End Main Container */
ods html close;
