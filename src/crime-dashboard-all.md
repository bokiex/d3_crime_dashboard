---
theme: dashboard
title: India Crime Statistics
toc: false
---

## India Crime Dashboard

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
const totalCases = d3.sum(yearData, (d) => d["TOTAL IPC CRIMES"]).toLocaleString();
const totalComplaints = d3.sum(yearData, (d) => d.police_population).toLocaleString();
const totalFirings = d3.sum(yearData, (d) => d.No_of_Firings).toLocaleString();
```

<div class="grid grid-cols-3">
  <div class="card">
    <h2>Total Crime Cases</h2>
    <span class="big">${totalCases}</span>
  </div>
  <div class="card">
    <h2>Total Police Strength</h2>
    <span class="big">${totalComplaints}</span>
  </div>
    <div class="card">
    <h2>Total Firings</h2>
    <span class="big">${totalFirings}</span>
  </div>
</div>

<!-- Police Conviction Rate by State -->

```js
function registeredRate(data, { width } = {}) {
    const filteredData = filterTop5(data);
    const stateRates = d3
        .rollups(
            filteredData,
            (v) => ({
                rate: d3.sum(v, (d) => d["TOTAL IPC CRIMES"]) / d3.sum(v, (d) => d.police_population),
                total: d3.sum(v, (d) => d["TOTAL IPC CRIMES"]),
                police: d3.sum(v, (d) => d.police_population),
            }),
            (d) => d.Area_Name
        )
        .sort((a, b) => b[1].rate - a[1].rate);

    return Plot.plot({
        title: "Crimes Committed",
        width,
        height: 500,
        marginLeft: 120,
        x: { label: "Number of crimes commited for every 1 policeman" },
        y: { label: null },
        marks: [
            Plot.barX(stateRates, {
                y: (d) => d[0],
                x: (d) => d[1].rate,
                fill: "steelblue",
                tip: true,
                title: (d) =>
                    `${d[0]}\nCrimes Committed: ${d[1].total}\nPolice Strength: ${d[1].police.toLocaleString()}`,
                sort: { y: "-x" },
            }),
        ],
    });
}
function casualtiesPerShot(data, { width } = {}) {
    const filteredData = filterTop5(data);
    const casualtyRates = d3
        .rollups(
            filteredData,
            (v) => {
                const totalCasualties = d3.sum(v, (d) => d.Civilians_Injured + d.Civilians_Killed);
                const totalFirings = d3.sum(v, (d) => d.No_of_Firings);
                return {
                    rate: totalFirings > 0 ? totalCasualties / totalFirings : 0,
                    casualties: totalCasualties,
                    firings: totalFirings,
                    civiliansKilled: d3.sum(v, (d) => d.Civilians_Killed),
                    civiliansInjured: d3.sum(v, (d) => d.Civilians_Injured),
                };
            },
            (d) => d.Area_Name
        )
        .sort((a, b) => b[1].rate - a[1].rate);

    return Plot.plot({
        title: "Casualties per Police Firing",
        width,
        height: 500,
        marginLeft: 120,
        x: {
            label: "Casualties per Firing",
            nice: true,
        },
        y: { label: null },
        marks: [
            Plot.barX(casualtyRates, {
                y: (d) => d[0],
                x: (d) => d[1].rate,
                fill: "red",
                tip: true,
                title: (d) =>
                    `${d[0]}\nCasualties per Firing: ${d[1].rate.toFixed(2)}\n` +
                    `Total Casualties: ${d[1].casualties}\n` +
                    `Civilians Killed: ${d[1].civiliansKilled}\n` +
                    `Civilians Injured: ${d[1].civiliansInjured}\n` +
                    `Total Firings: ${d[1].firings}`,
                sort: { y: "-x" },
            }),
        ],
    });
}

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

function crimeAndPoliceTrends(data, { width } = {}) {
    const filteredData = crimeData;

    // Aggregate data by year
    const yearlyData = d3
        .rollups(
            filteredData,
            (v) => ({
                totalCrimes: d3.sum(v, (d) => d["TOTAL IPC CRIMES"]),
                policeStrength: d3.sum(v, (d) => d.police_population),
                crimeRate: d3.sum(v, (d) => d["TOTAL IPC CRIMES"]) / d3.sum(v, (d) => d.police_population),
            }),
            (d) => d.Year
        )
        .map(([year, stats]) => ({
            Year: year,
            ...stats,
        }));

    // Sort by year
    yearlyData.sort((a, b) => a.Year - b.Year);

    return Plot.plot({
        title: "Crime Cases and Police Strength Trends",
        width,
        height: 500,
        marginTop: 30,
        marginLeft: 60,
        marginRight: 60,
        x: {
            label: "Year",
            type: "linear",
            nice: true,
        },
        y: {
            label: "Number of Crime Cases",
            type: "linear",
            nice: true,
            grid: true,
        },
        y2: {
            label: "Police Strength",
            type: "linear",
            nice: true,
            grid: false,
        },
        color: {
            legend: true,
            domain: ["Crime Cases", "Police Strength"],
            range: ["#e41a1c", "#377eb8"],
        },
        marks: [
            // Line for crime cases
            Plot.line(yearlyData, {
                x: "Year",
                y: "totalCrimes",
                stroke: "#e41a1c",
                strokeWidth: 2,
                tip: true,
                title: (d) =>
                    `Year: ${
                        d.Year
                    }\nCrime Cases: ${d.totalCrimes.toLocaleString()}\nPolice Strength: ${d.policeStrength.toLocaleString()}\nCrime Rate: ${d.crimeRate.toFixed(
                        2
                    )} per police officer`,
            }),
            // Dots for crime cases
            Plot.dot(yearlyData, {
                x: "Year",
                y: "totalCrimes",
                fill: "#e41a1c",
                r: 4,
                tip: true,
                title: (d) =>
                    `Year: ${
                        d.Year
                    }\nCrime Cases: ${d.totalCrimes.toLocaleString()}\nPolice Strength: ${d.policeStrength.toLocaleString()}\nCrime Rate: ${d.crimeRate.toFixed(
                        2
                    )} per police officer`,
            }),
            // Line for police strength
            Plot.line(yearlyData, {
                x: "Year",
                y: "policeStrength",
                stroke: "#377eb8",
                strokeWidth: 2,
                y2: true,
                tip: true,
                title: (d) =>
                    `Year: ${
                        d.Year
                    }\nCrime Cases: ${d.totalCrimes.toLocaleString()}\nPolice Strength: ${d.policeStrength.toLocaleString()}\nCrime Rate: ${d.crimeRate.toFixed(
                        2
                    )} per police officer`,
            }),
            // Dots for police strength
            Plot.dot(yearlyData, {
                x: "Year",
                y: "policeStrength",
                fill: "#377eb8",
                r: 4,
                y2: true,
                tip: true,
                title: (d) =>
                    `Year: ${
                        d.Year
                    }\nCrime Cases: ${d.totalCrimes.toLocaleString()}\nPolice Strength: ${d.policeStrength.toLocaleString()}\nCrime Rate: ${d.crimeRate.toFixed(
                        2
                    )} per police officer`,
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
    ${resize((width) => casualtiesPerShot(crimeData, {width}))}
  </div>
</div>

<div class="grid grid-cols-2">
  <div class="card">
    ${resize((width) => crimeTypesOverTime(crimeData, {width}))}
  </div>
    <div class="card">
    ${resize((width) => crimeAndPoliceTrends(crimeData, {width}))}
  </div>
</div>
