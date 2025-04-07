---
theme: dashboard
title: India Police Effectiveness
toc: false
---

# India Police Effectiveness Dashboard 🚨

<!-- Load and transform the data -->

```js
// Load the data and handle the Promise
const crimeData = await FileAttachment("data/main_df.csv").csv({ typed: true });
console.log(crimeData);

// Create year range for filter from 2001-2010 with proper stepping
const years = [2001, 2010];
const yearFilter = view(
    Inputs.range(years, {
        label: "Select Year",
        value: years[0],
        step: 1,
    })
);

// Add checkbox for top 5 filter
const showTop5 = view(
    Inputs.checkbox({
        label: "Show Only Top 5 States",
        value: false,
    })
);

// Helper function to get top 5 states
function getTop5States(data) {
    const stateCases = d3.rollup(
        data,
        (v) => d3.sum(v, (d) => d.cases_registered_against_police),
        (d) => d.Area_Name
    );
    return Array.from(stateCases.entries())
        .sort((a, b) => b[1] - a[1])
        .slice(0, 5)
        .map(([state]) => state);
}

// Helper function to filter data based on top 5
function filterTop5(data) {
    if (!showTop5) return data;
    const top5States = getTop5States(data);
    return data.filter((d) => top5States.includes(d.Area_Name));
}
```

<!-- Summary Cards -->

```js
// Filter data by selected year and top 5 if needed
const yearData = filterTop5(crimeData.filter((d) => d.Year === yearFilter));
// Create a color scale for states
const color = Plot.scale({
    color: {
        type: "categorical",
        domain: d3.groupSort(
            yearData,
            (d) => -d.cases_registered_against_police,
            (d) => d.Area_Name
        ),
        unknown: "var(--theme-foreground-muted)",
    },
});
const totalCases = d3.sum(yearData, (d) => d.cases_registered_against_police).toLocaleString();
const totalComplaints = d3.sum(yearData, (d) => d.complaints_received_against_police).toLocaleString();
const totalArrests = d3.sum(yearData, (d) => d.police_personnel_sent_for_trial).toLocaleString();
const totalConvictions = d3.sum(yearData, (d) => d.police_personnel_convicted).toLocaleString();
```

<div class="grid grid-cols-4">
  <div class="card">
    <h2>Total Cases Against Police</h2>
    <span class="big">${totalCases}</span>
  </div>
  <div class="card">
    <h2>Total Complaints Against Police</h2>
    <span class="big">${totalComplaints}</span>
  </div>
  <div class="card">
    <h2>Total Police Sent for Trial</h2>
    <span class="big">${totalArrests}</span>
  </div>
  <div class="card">
    <h2>Total Policemen Convicted</h2>
    <span class="big">${totalConvictions}</span>
  </div>
</div>

<!-- Police Conviction Rate by State -->

```js
function registeredRate(data, { width } = {}) {
    const filteredData = filterTop5(crimeData.filter((d) => d.Year === yearFilter));
    const stateRates = d3
        .rollups(
            filteredData,
            (v) => ({
                rate: d3.sum(v, (d) => d.cases_registered_against_police) / d3.sum(v, (d) => d.police_population),
                total: d3.sum(v, (d) => d.cases_registered_against_police),
                police: d3.sum(v, (d) => d.police_population),
            }),
            (d) => d.Area_Name
        )
        .sort((a, b) => b[1].rate - a[1].rate);

    return Plot.plot({
        title: "Cases Registered Rate against Police by State (%)",
        width,
        height: 500,
        marginLeft: 120,
        x: { label: "Conviction Rate (%)" },
        y: { label: null },
        marks: [
            Plot.barX(stateRates, {
                y: (d) => d[0],
                x: (d) => d[1].rate,
                fill: "steelblue",
                tip: true,
                title: (d) => `${d[0]}\nConvicted: ${d[1].total}\nPolice Force: ${d[1].police.toLocaleString()}`,
                sort: { y: "-x" },
            }),
        ],
    });
}
// Case Disposition Analysis

function caseDisposition(data, { width } = {}) {
    const filteredData = filterTop5(crimeData.filter((d) => d.Year === yearFilter));
    const dispositionData = d3.rollups(
        filteredData,
        (v) => ({
            sentToTrial: d3.sum(v, (d) => d.police_personnel_sent_for_trial),
            falseOrWithdrawn: d3.sum(
                v,
                (d) =>
                    d.complaints_or_cases_declared_false_or_unsubstantiated_against_police +
                    d.police_personnel_cases_withdrawn_or_disposed_of
            ),
        }),
        (d) => d.Area_Name
    );

    return Plot.plot({
        title: "Case Disposition Analysis",
        width,
        height: 500,
        marginLeft: 120,
        x: { label: "Number of Cases" },
        y: { label: null },
        color: {
            legend: true,
            domain: ["Cases Registered", "Cases Withdrawn"],
            range: ["#1f77b4", "#ff7f0e"],
        },
        marks: [
            Plot.barX(dispositionData, {
                y: (d) => d[0],
                x: (d) => d[1].sentToTrial,
                fill: "steelblue",
                tip: true,
                title: (d) => `${d[0]}\nSent to Trial: ${d[1].sentToTrial}\nFalse/Withdrawn: ${d[1].falseOrWithdrawn}`,
                sort: { y: "-x" },
            }),
            Plot.barX(dispositionData, {
                y: (d) => d[0],
                x: (d) => -d[1].falseOrWithdrawn,
                fill: "red",
                tip: true,
                title: (d) => `${d[0]}\nSent to Trial: ${d[1].sentToTrial}\nFalse/Withdrawn: ${d[1].falseOrWithdrawn}`,
            }),
        ],
    });
}
```

<div class="grid grid-cols-2">
  <div class="card">
    ${resize((width) => registeredRate(crimeData, {width}))}
  </div>
  <div class="card">
    ${resize((width) => caseDisposition(crimeData, {width}))}
  </div>
</div>

<!-- Trends Over Time -->

```js
function trendsOverTime(data, { width } = {}) {
    const filteredData = showTop5 ? filterTop5(crimeData) : crimeData;
    return Plot.plot({
        title: "Police Cases Registered Over Time",
        width,
        height: 400,
        marginTop: 30,
        marginLeft: 60,
        x: { label: "Year" },
        y: { label: "Number of Cases" },
        // check if the
        color: { ...color, legend: true },
        marks: [
            Plot.line(filteredData, {
                x: "Year",
                y: "cases_registered_against_police",
                stroke: "Area_Name",
                strokeWidth: 2,
                tip: true,
            }),
            Plot.dot(filteredData, {
                x: "Year",
                y: "cases_registered_against_police",
                fill: "Area_Name",
                r: 4,
                tip: true,
            }),
        ],
    });
}

function casesOverTime(data, { width } = {}) {
    // Aggregate data by year
    const yearlyData = d3
        .rollups(
            data,
            (v) => ({
                registered: d3.sum(v, (d) => d.cases_registered_against_police),
                withdrawn: d3.sum(v, (d) => d.police_personnel_cases_withdrawn_or_disposed_of),
                trialCompleted: d3.sum(v, (d) => d.police_cases_trial_completed),
            }),
            (d) => d.Year
        )
        .map(([year, stats]) => ({
            Year: year,
            registered: stats.registered,
            withdrawn: stats.withdrawn,
            trialCompleted: stats.trialCompleted,
        }));

    // Sort data by year
    yearlyData.sort((a, b) => a.Year - b.Year);

    console.log("Yearly data:", yearlyData); // Debug log

    return Plot.plot({
        title: "Police Case Statistics Over Time",
        width,
        height: 400,
        marginTop: 30,
        marginLeft: 60,
        x: {
            label: "Year",
            type: "linear",
            nice: true,
        },
        y: {
            label: "Number of Cases",
            type: "linear",
            nice: true,
        },
        color: {
            legend: true,
            domain: ["Cases Registered", "Cases Withdrawn", "Trial Completed"],
            range: ["#1f77b4", "#ff7f0e", "#2ca02c"],
        },
        marks: [
            Plot.line(yearlyData, {
                x: "Year",
                y: "registered",
                stroke: "#1f77b4",
                strokeWidth: 2,
                tip: true,
                title: (d) => `Cases Registered: ${d.registered.toLocaleString()}`,
            }),
            Plot.line(yearlyData, {
                x: "Year",
                y: "withdrawn",
                stroke: "#ff7f0e",
                strokeWidth: 2,
                tip: true,
                title: (d) => `Cases Withdrawn: ${d.withdrawn.toLocaleString()}`,
            }),
            Plot.line(yearlyData, {
                x: "Year",
                y: "trialCompleted",
                stroke: "#2ca02c",
                strokeWidth: 2,
                tip: true,
                title: (d) => `Trial Completed: ${d.trialCompleted.toLocaleString()}`,
            }),
            Plot.dot(yearlyData, {
                x: "Year",
                y: "registered",
                fill: "#1f77b4",
                r: 4,
                tip: true,
                title: (d) => `Cases Registered: ${d.registered.toLocaleString()}`,
            }),
            Plot.dot(yearlyData, {
                x: "Year",
                y: "withdrawn",
                fill: "#ff7f0e",
                r: 4,
                tip: true,
                title: (d) => `Cases Withdrawn: ${d.withdrawn.toLocaleString()}`,
            }),
            Plot.dot(yearlyData, {
                x: "Year",
                y: "trialCompleted",
                fill: "#2ca02c",
                r: 4,
                tip: true,
                title: (d) => `Trial Completed: ${d.trialCompleted.toLocaleString()}`,
            }),
        ],
    });
}
```

<div class="grid grid-cols-2">
  <div class="card">
    ${resize((width) => trendsOverTime(crimeData, {width}))}
  </div>
  
  <div class="card">
    ${resize((width) => casesOverTime(crimeData, {width}))}
  </div>
</div>
