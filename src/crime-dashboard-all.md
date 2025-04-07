---
theme: dashboard
title: India Crime Statistics
toc: false
---

## India Overall Dashboard

## India Police Dashboard 🚨

<!-- Load and transform the data -->

```js
// Load the data and handle the Promise
const crimeData = await FileAttachment("data/main_df.csv").csv({ typed: true });
console.log(crimeData);

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
const yearData = filterTop5(crimeData);
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
    const filteredData = filterTop5(crimeData);
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
    const filteredData = filterTop5(crimeData);
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
            domain: ["Cases Registered", "Cases Withdrawn", "Trial Completed"],
            range: ["#1f77b4", "#ff7f0e", "#2ca02c"],
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

```js
function crimeTypesOverTime(data, { width } = {}) {
    const filteredData = crimeData;

    // Define major crime categories
    const crimeCategories = [
        { key: "MURDER", label: "Murder", color: "#e41a1c" },
        { key: "RAPE", label: "Rape", color: "#377eb8" },
        { key: "KIDNAPPING & ABDUCTION", label: "Kidnapping", color: "#4daf4a" },
        { key: "ROBBERY", label: "Robbery", color: "#984ea3" },
        { key: "BURGLARY", label: "Burglary", color: "#ff7f00" },
        { key: "THEFT", label: "Theft", color: "#ffff33" },
        { key: "RIOTS", label: "Riots", color: "#a65628" },
        { key: "HURT/GREVIOUS HURT", label: "Grievous Hurt", color: "#f781bf" },
    ];

    // Transform data for line chart
    const yearlyData = d3
        .rollups(
            filteredData,
            (v) => {
                const stats = {};
                crimeCategories.forEach((cat) => {
                    stats[cat.key] = d3.sum(v, (d) => d[cat.key]);
                });
                return stats;
            },
            (d) => d.Year
        )
        .map(([year, stats]) => ({
            Year: year,
            ...stats,
        }));

    // Sort by year
    yearlyData.sort((a, b) => a.Year - b.Year);

    // Create marks for each crime type
    const marks = crimeCategories.map((cat) =>
        Plot.line(yearlyData, {
            x: "Year",
            y: cat.key,
            stroke: cat.color,
            strokeWidth: 2,
            tip: true,
            title: (d) => `${cat.label}: ${d[cat.key].toLocaleString()}`,
        })
    );

    // Add dots for each data point
    const dots = crimeCategories.map((cat) =>
        Plot.dot(yearlyData, {
            x: "Year",
            y: cat.key,
            fill: cat.color,
            r: 4,
            tip: true,
            title: (d) => `${cat.label}: ${d[cat.key].toLocaleString()}`,
        })
    );

    return Plot.plot({
        title: "Major Crime Types Over Time",
        width,
        height: 500,
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
            domain: crimeCategories.map((c) => c.label),
            range: crimeCategories.map((c) => c.color),
        },
        marks: [...marks, ...dots],
    });
}

function crimeTypeDistribution(data, { width } = {}) {
    const filteredData = crimeData;

    // Define crime categories with their colors
    const crimeCategories = [
        { key: "MURDER", label: "Murder", color: "#e41a1c" },
        { key: "RAPE", label: "Rape", color: "#377eb8" },
        { key: "KIDNAPPING & ABDUCTION", label: "Kidnapping", color: "#4daf4a" },
        { key: "ROBBERY", label: "Robbery", color: "#984ea3" },
        { key: "BURGLARY", label: "Burglary", color: "#ff7f00" },
        { key: "THEFT", label: "Theft", color: "#ffff33" },
        { key: "RIOTS", label: "Riots", color: "#a65628" },
        { key: "HURT/GREVIOUS HURT", label: "Grievous Hurt", color: "#f781bf" },
    ];

    // Aggregate data by crime type
    const crimeData = crimeCategories.map((cat) => ({
        name: cat.label,
        value: d3.sum(filteredData, (d) => d[cat.key]),
    }));

    // Sort by value
    crimeData.sort((a, b) => b.value - a.value);

    // Calculate total for percentage
    const total = d3.sum(crimeData, (d) => d.value);

    return Plot.plot({
        title: "Crime Type Distribution",
        width,
        height: 500,
        marginTop: 30,
        marginLeft: 60,
        color: {
            legend: true,
            domain: crimeCategories.map((c) => c.label),
            range: crimeCategories.map((c) => c.color),
        },
        marks: [
            Plot.arc(crimeData, {
                innerRadius: 100,
                outerRadius: 200,
                padAngle: 0.01,
                fill: "name",
                title: (d) =>
                    `${d.name}\nCases: ${d.value.toLocaleString()}\nPercentage: ${((d.value / total) * 100).toFixed(
                        1
                    )}%`,
                tip: true,
            }),
        ],
    });
}
```

<div class="grid grid-cols-2">
  <div class="card">
    ${resize((width) => crimeTypesOverTime(crimeData, {width}))}
  </div>

  <div class="card">
    ${resize((width) => crimeTypeDistribution(crimeData, {width}))}
  </div>
</div>
