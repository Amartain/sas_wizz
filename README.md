# sas_wizz
Sas html viz. scripts, mostly gemini, to start out

/* ================================================================================= */
/* PART A: DEFINE "SEABORN MUTED" STYLE                                              */
/* ================================================================================= */
proc template;
   define style styles.SeabornMuted;
      parent = styles.htmlblue; /* Inherit basics */
      
      /* 1. Muted Color Palette (Blue, Green, Red, Purple, Yellow, Cyan) */
      class GraphData1 / contrastcolor=#4C72B0 color=#4C72B0; /* Muted Blue */
      class GraphData2 / contrastcolor=#55A868 color=#55A868; /* Muted Green */
      class GraphData3 / contrastcolor=#C44E52 color=#C44E52; /* Muted Red */
      class GraphData4 / contrastcolor=#8172B2 color=#8172B2; /* Muted Purple */
      class GraphData5 / contrastcolor=#CCB974 color=#CCB974; /* Muted Yellow */
      
      /* 2. Backgrounds & Grids (Seaborn style is usually light grey grid on white) */
      class GraphWalls / frameborder=off backgroundcolor=white;
      class GraphBackground / backgroundcolor=white;
      class GraphGridLines / displayopts=auto contrastcolor=#EAEAF2 line_style=1; 
      
      /* 3. Fonts (Clean sans-serif) */
      class GraphFonts /
         'GraphDataFont' = ("Segoe UI, Helvetica, Arial", 9pt)
         'GraphValueFont' = ("Segoe UI, Helvetica, Arial", 10pt)
         'GraphLabelFont' = ("Segoe UI, Helvetica, Arial", 11pt, bold)
         'GraphTitleFont' = ("Segoe UI, Helvetica, Arial", 14pt, bold);
   end;
run;

/* ================================================================================= */
/* PART B: CREATE MOCK DATA TABLES                                                   */
/* ================================================================================= */

/* Table 1: Department Stats */
data work.dep_stats;
    input DepName $ DepID $ No_Cases Avg_Satisfaction;
    datalines;
Sales D01 150 4.5
IT    D02 85  3.2
HR    D03 40  4.8
;
run;

/* Table 2: Employee Stats */
data work.emp_stats;
    input EmpName $ EmpID $ DepID $ No_Cases Total_Hours;
    datalines;
Adam  E01 D01 10 120
Eve   E02 D01 15 160
Steve E03 D02 8  90
Sarah E04 D02 12 110
John  E05 D03 5  40
;
run;

/* Table 3: CORE DATA (Transactions) */
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
;
run;

/* ================================================================================= */
/* PART C: PREPARE MACRO VARIABLES FOR DROPDOWNS                                     */
/* ================================================================================= */
proc sql noprint;
    select distinct DepName into :dept_list separated by '|' from work.dep_stats;
    select distinct EmpName into :emp_list separated by '|' from work.emp_stats;
quit;

%put NOTE: Data Preparation Complete. Styles and Tables Ready.;



# Dashboard generator

/* ================================================================================= */
/* CONFIGURATION: OUTPUT LOCATION                                                    */
/* ================================================================================= */
%let outpath = C:\temp;
%let filename = Seaborn_Dashboard.html;

/* Close any open destinations */
ods _all_ close;

/* ================================================================================= */
/* GENERATE HTML REPORT USING "SeabornMuted" STYLE                                   */
/* ================================================================================= */
ods html file="&outpath\&filename" path="&outpath" style=styles.SeabornMuted
    headtext='
    <link rel="stylesheet" type="text/css" href="https://cdn.datatables.net/1.10.20/css/jquery.dataTables.css">
    <script src="https://code.jquery.com/jquery-3.3.1.js"></script>
    <script src="https://cdn.datatables.net/1.10.20/js/jquery.dataTables.js"></script>
    
    <style>
        body { font-family: "Segoe UI", sans-serif; background: #f0f0f0; padding: 25px; color: #444; }
        
        /* Navigation Bar */
        .tab-nav { overflow: hidden; background: #333; border-radius: 4px 4px 0 0; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .tab-nav button { background: inherit; float: left; border: none; cursor: pointer; padding: 14px 20px; color: #ccc; font-size: 15px; transition: 0.2s; }
        .tab-nav button:hover { background: #444; color: white; }
        .tab-nav button.active { background: #4C72B0; color: white; font-weight: 600; } /* Matches Seaborn Blue */
        
        /* Tab Content Area */
        .tab-content { display: none; padding: 25px; background: white; border: 1px solid #ddd; border-top: none; box-shadow: 0 2px 5px rgba(0,0,0,0.05); }
        
        /* Dropdown Controls */
        .control-panel { background: #f8f9fa; padding: 15px; border-radius: 4px; border-left: 4px solid #4C72B0; margin-bottom: 20px; }
        select { padding: 8px; border: 1px solid #ccc; border-radius: 3px; font-size: 14px; min-width: 200px; }
        
        /* Graphs */
        .graph-container { display: none; animation: fadeIn 0.4s; }
        @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
        
        h1 { color: #333; margin-bottom: 5px; }
        h3 { color: #4C72B0; border-bottom: 2px solid #eee; padding-bottom: 10px; margin-top: 30px; }
    </style>

    <script>
        function openTab(evt, tabName) {
            $(".tab-content").hide();
            $(".tablinks").removeClass("active");
            $("#" + tabName).show();
            $(evt.currentTarget).addClass("active");
        }
        function filterDept(val) { $(".dept-drill").hide(); $("#D_"+val).show(); }
        function filterEmp(val) { $(".emp-drill").hide(); $("#E_"+val).show(); }
        
        $(document).ready(function() {
            $(".search-table").DataTable({ "pageLength": 10 });
            document.getElementById("defaultOpen").click();
        });
    </script>
    ';

/* --- HEADER --- */
ods html text='<h1>Strategic Analysis Dashboard</h1>';
ods html text='<p style="color:#777; margin-bottom:20px;">Interactive Reporting System | <i>Generated by SAS Enterprise Guide</i></p>';

/* --- NAVIGATION --- */
ods html text='
<div class="tab-nav">
  <button class="tablinks" onclick="openTab(event, ''TabDept'')" id="defaultOpen">Department Stats</button>
  <button class="tablinks" onclick="openTab(event, ''TabEmp'')">Employee Stats</button>
  <button class="tablinks" onclick="openTab(event, ''TabData'')">Data Vault</button>
</div>
';

/* ============================================================ */
/* TAB 1: DEPARTMENT ANALYSIS                                   */
/* ============================================================ */
ods html text='<div id="TabDept" class="tab-content">';
    
    /* Overview Chart */
    ods html text='<h3>Departmental Overview</h3>';
    proc sgplot data=work.dep_stats;
        /* Using the Seaborn style colors automatically */
        vbar DepName / response=No_Cases group=DepName;
        yaxis grid;
        title "Total Case Volume by Department";
    run;
    
    /* Drill-Down Section */
    ods html text='<h3>Detailed Breakdown</h3>';
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
                title "Case Types: &d";
                proc sgplot data=work.core_data;
                    where DepName = "&d";
                    /* Horizontal bar, slightly transparent like Seaborn */
                    hbar CaseType / response=Indiv_Hours fillattrs=(transparency=0.2); 
                    xaxis grid label="Hours Invested";
                run;
            ods html text="</div>";
        %end;
    %mend;
    %dept_drill;

ods html text='</div>';

/* ============================================================ */
/* TAB 2: EMPLOYEE ANALYSIS                                     */
/* ============================================================ */
ods html text='<div id="TabEmp" class="tab-content">';

    /* Overview Chart */
    ods html text='<h3>Top Performers</h3>';
    proc sgplot data=work.emp_stats;
        /* Sorted bar chart */
        vbar EmpName / response=No_Cases categoryorder=respdesc fillattrs=(color='#4C72B0');
        yaxis grid;
        title "Cases Closed (Leaderboard)";
    run;

    /* Drill-Down Section */
    ods html text='<h3>Individual Workload Analysis</h3>';
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
                title "Work Distribution: &e";
                proc sgplot data=work.core_data;
                    where EmpName = "&e";
                    /* Compare with transparency */
                    vbar CaseID / response=Indiv_Hours name="Mine" legendlabel="User Hours" fillattrs=(color='#55A868');
                    vbar CaseID / response=Full_Case_Hours name="Full" legendlabel="Total Case Hours" fillattrs=(color='#444444' transparency=0.8);
                    keylegend "Mine" "Full" / location=inside position=topright;
                    yaxis grid;
                run;
            ods html text="</div>";
        %end;
    %mend;
    %emp_drill;

ods html text='</div>';

/* ============================================================ */
/* TAB 3: DATA VAULT                                            */
/* ============================================================ */
ods html text='<div id="TabData" class="tab-content">';
    ods html text='<h3>Master Transaction Database</h3>';
    ods html text='<p>Full searchable access to the <i>Core Data</i> repository.</p>';

    /* Using DataTables */
    proc print data=work.core_data noobs label 
        style(table)={htmlclass="search-table" width="100%" frame=box rules=rows};
    run;

ods html text='</div>';

ods html close;



# 
