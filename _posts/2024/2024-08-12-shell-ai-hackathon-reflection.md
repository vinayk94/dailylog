---
layout: post
title: "Reflections on the Shell.ai Hackathon 2024"
date: 2024-08-12
categories: [hackathon, optimization, sustainability]
---

Recently, I participated in the Shell.ai Hackathon for Sustainable and Affordable Energy 2024. The challenge focused on optimizing fleet decarbonization strategies, a crucial step towards achieving net-zero emissions in the transportation sector.  This post is my attempt to document my approach.


## The Problem

The hackathon presented us with a complex real-world problem: how to transition a fleet of vehicles to lower-emission alternatives while balancing operational costs, meeting demand, and adhering to increasingly strict emission limits over a 16-year period (2023-2038).

Key aspects of the problem included:
- Multiple vehicle types (Diesel, LNG, BEV) with different characteristics
- Varying fuel types, costs, and associated emissions
- Yearly demand that must be satisfied for different vehicle sizes and distance requirements
- Carbon emission limits that decrease over time
- Vehicle lifecycle constraints (10-year lifespan, purchase/sell timing)

The challenge was not just about finding the cheapest or lowest-emission solution, but about finding an optimal balance that could realistically be implemented by fleet operators.


## Linear Programming and PuLP

After analyzing the problem, I decided to approach it using linear programming. Here's why:

1. The problem involves optimizing a linear objective (total cost) subject to linear constraints (demand satisfaction, emission limits, etc.).
2. Decision variables (number of vehicles to buy, use, or sell) are inherently integer-valued, making it a mixed-integer linear programming (MILP) problem.
3. Linear programming is well-suited for problems with many variables and constraints, which was the case here with multiple years, vehicle types, and operational constraints.

I chose PuLP as the tool to implement this linear programming model as its an open-source library to quickly make protypes with Python.


## Decision Variables

The problem revolves around three key decisions: buying, using, and selling vehicles. Therefore, I defined the following decision variables:

1. \(buy_{y,v}\): Number of vehicles of type \(v\) bought in year \(y\)
2. \(use_{y,v}\): Number of vehicles of type \(v\) used in year \(y\)
3. \(sell_{y,v}\): Number of vehicles of type \(v\) sold in year \(y\)

These variables are crucial as they allow us to track the fleet's composition and usage over time. They directly represent the decisions a fleet manager would make each year.

In PuLP, these variables are implemented as follows:

```python
buy_vars = LpVariable.dicts("buy", 
                            [(year, v_id) for year in range(2023, 2039) for v_id in vehicles['ID']], 
                            lowBound=0, cat=LpInteger)

use_vars = LpVariable.dicts("use", 
                            [(year, v_id) for year in range(2023, 2039) for v_id in vehicles['ID']], 
                            lowBound=0, cat=LpInteger)

sell_vars = LpVariable.dicts("sell", 
                             [(year, v_id) for year in range(2023, 2039) for v_id in vehicles['ID']], 
                             lowBound=0, cat=LpInteger)
```
## Objective Function

The objective is to minimize the total cost over the planning horizon. This includes several cost components:

\[ \text{Minimize } C_{total} = \sum_{y=2023}^{2038} (C_{buy}^y + C_{ins}^y + C_{mnt}^y + C_{fuel}^y - C_{sell}^y) \]

Let's break down each cost component:

### 1. Buying Costs

Buying costs represent the capital expenditure for fleet renewal. This is crucial because transitioning to lower-emission vehicles often requires significant upfront investment.

\[ C_{buy}^y = \sum_{v \in V} buy_{y,v} \cdot cost_v \]

In code:

```python
buying_costs = lpSum([
    buy_vars[(year, v_id)] * vehicles.loc[vehicles['ID'] == v_id, 'Cost ($)'].values[0]
    for year in range(2023, 2039)
    for v_id in vehicles['ID']
])
```

### 2. Insurance Costs

Insurance is an ongoing operational cost that varies with the age and value of vehicles. Including it ensures we account for the full cost of ownership.

\[ C_{ins}^y = \sum_{v \in V} fleet_{y,v} \cdot cost_v \cdot ins_{y-y_p}^v \]

Where \(fleet_{y,v}\) is the number of vehicles of type \(v\) in the fleet in year \(y\), and \(ins_{y-y_p}^v\) is the insurance cost percentage for a vehicle of age \(y-y_p\).

In code:

```python
insurance_costs = lpSum([
    fleet_size[(year, v_id)] * vehicles.loc[vehicles['ID'] == v_id, 'Cost ($)'].values[0] *
    get_cost_profile_value(year, int(v_id.split('_')[-1]), 'Insurance Cost %')
    for year in range(2023, 2039)
    for v_id in vehicles['ID']
])
```

### 3. Maintenance Costs

Maintenance costs typically increase as vehicles age. This component encourages the model to balance between keeping older, fully depreciated vehicles and investing in new, potentially more efficient ones.

\[ C_{mnt}^y = \sum_{v \in V} fleet_{y,v} \cdot cost_v \cdot mnt_{y-y_p}^v \]

Where \(mnt_{y-y_p}^v\) is the maintenance cost percentage for a vehicle of age \(y-y_p\).

In code:

```python
maintenance_costs = lpSum([
    fleet_size[(year, v_id)] * vehicles.loc[vehicles['ID'] == v_id, 'Cost ($)'].values[0] *
    get_cost_profile_value(year, int(v_id.split('_')[-1]), 'Maintenance Cost %')
    for year in range(2023, 2039)
    for v_id in vehicles['ID']
])
```

### 4. Fuel Costs

Fuel costs are a major part of operational expenses and directly tied to emissions. This component drives the model to consider more fuel-efficient or alternative fuel vehicles.

\[ C_{fuel}^y = \sum_{v \in V} use_{y,v} \cdot cons_v \cdot fuel\_cost_{y,f_v} \cdot range_v \]

Where \(cons_v\) is the fuel consumption of vehicle \(v\), \(fuel\_cost_{y,f_v}\) is the cost of fuel for vehicle \(v\) in year \(y\), and \(range_v\) is the yearly range of the vehicle.

In code:

```python
fuel_costs = lpSum([
    use_vars[(year, v_id)] * vehicles_fuels.loc[vehicles_fuels['ID'] == v_id, 'Consumption (unit_fuel/km)'].values[0] *
    fuels.loc[(fuels['Fuel'] == vehicles_fuels.loc[vehicles_fuels['ID'] == v_id, 'Fuel'].values[0]) & (fuels['Year'] == year), 'Cost ($/unit_fuel)'].values[0] *
    vehicles.loc[vehicles['ID'] == v_id, 'Yearly range (km)'].values[0]
    for year in range(2023, 2039)
    for v_id in vehicles['ID']
    if (year, v_id) in use_vars
])
```

### 5. Resale Value

Including resale value encourages the model to optimize the timing of vehicle replacements, balancing the benefits of keeping fully depreciated vehicles against the advantages of newer, potentially more efficient models.

\[ C_{sell}^y = \sum_{v \in V} sell_{y,v} \cdot cost_v \cdot resale_{y-y_p}^v \]

Where \(resale_{y-y_p}^v\) is the resale value percentage for a vehicle of age \(y-y_p\).

In code:

```python
resale_value = lpSum([
    sell_vars[(year, v_id)] * vehicles.loc[vehicles['ID'] == v_id, 'Cost ($)'].values[0] *
    get_cost_profile_value(year, int(v_id.split('_')[-1]), 'Resale Value %')
    for year in range(2023, 2039)
    for v_id in vehicles['ID']
    if (year, v_id) in sell_vars
])
```

## Constraints

### 1. Demand Satisfaction

This constraint ensures that the fleet can meet all transportation demands each year, which is crucial for maintaining business operations.

\[ \sum_{v \in V_{s,d}} use_{y,v} \cdot range_v \geq demand_{y,s,d} \quad \forall y,s,d \]

Where \(V_{s,d}\) is the set of vehicles of size \(s\) that can cover distance \(d\).

In code:

```python
for _, row in demand.iterrows():
    model += lpSum([
        use_vars[(row['Year'], v_id)] * vehicles.loc[vehicles['ID'] == v_id, 'Yearly range (km)'].values[0]
        for v_id in vehicles['ID']
        if (row['Year'], v_id) in use_vars and
           vehicles.loc[vehicles['ID'] == v_id, 'Size'].values[0] == row['Size'] and 
           vehicles.loc[vehicles['ID'] == v_id, 'Distance'].values[0] >= row['Distance']
    ]) >= row['Demand (km)']
```

### 2. Carbon Emissions

This constraint drives the decarbonization aspect of the problem, forcing the model to transition to lower-emission vehicles over time.

\[ \sum_{v \in V} use_{y,v} \cdot cons_v \cdot emission_{f_v} \cdot range_v \leq emission\_limit_y \quad \forall y \]

Where \(emission_{f_v}\) is the emission factor for the fuel used by vehicle \(v\).

In code:

```python
for year in carbon_emissions['Year']:
    model += lpSum([
        use_vars[(year, v_id)] * vehicles_fuels.loc[vehicles_fuels['ID'] == v_id, 'Consumption (unit_fuel/km)'].values[0] * 
        fuels.loc[(fuels['Fuel'] == vehicles_fuels.loc[vehicles_fuels['ID'] == v_id, 'Fuel'].values[0]) & (fuels['Year'] == year), 'Emissions (CO2/unit_fuel)'].values[0] * 
        vehicles.loc[vehicles['ID'] == v_id, 'Yearly range (km)'].values[0]
        for v_id in vehicles['ID']
        if (year, v_id) in use_vars
    ]) <= carbon_emissions.loc[carbon_emissions['Year'] == year, 'Carbon emission CO2/kg'].values[0]
```

### 3. Vehicle Lifecycle

These constraints model the 10-year lifecycle of vehicles, ensuring they are only bought in their designated purchase year and sold by the end of their useful life.

\[ use_{y,v} = sell_{y,v} = 0, \quad \forall y < y_p \text{ or } y \geq y_p + 10 \]
\[ buy_{y,v} = 0, \quad \forall y \neq y_p \]
\[ \sum_{y=y_p}^{y_p+9} sell_{y,v} = \sum_{y=y_p}^{y_p+9} buy_{y,v} \quad \forall v \]

In code:

```python
for v_id in vehicles['ID']:
    purchase_year = int(v_id.split('_')[-1])
    for year in range(2023, 2039):
        if year < purchase_year or year >= purchase_year + 10:
            model += use_vars[(year, v_id)] == 0
            model += sell_vars[(year, v_id)] == 0

        if year != purchase_year:
            model += buy_vars[(year, v_id)] == 0

    model += lpSum([sell_vars[(year, v_id)] for year in range(purchase_year, purchase_year + 10) if (year, v_id) in sell_vars]) == \
             lpSum([buy_vars[(year, v_id)] for year in range(purchase_year, purchase_year + 10) if (year, v_id) in buy_vars])
```

### 4. Fleet Size Calculation

This constraint keeps track of the fleet composition, which is necessary for calculating insurance and maintenance costs.

\[ fleet_{y,v} = \sum_{i=y_p}^{y} (buy_{i,v} - sell_{i,v}) \quad \forall y,v \]

In code:

```python
fleet_size = {}
for year in range(2023, 2039):
    for v_id in vehicles['ID']:
        purchase_year = int(v_id.split('_')[-1])
        if purchase_year <= year < purchase_year + 10:
            fleet_size[(year, v_id)] = (
                lpSum([buy_vars[(y, v_id)] for y in range(purchase_year, year + 1)]) - 
                lpSum([sell_vars[(y, v_id)] for y in range(purchase_year, year + 1) if (y, v_id) in sell_vars])
            )
        else:
            fleet_size[(year, v_id)] = 0
```

## Solution and Analysis

The model was solved using PuLP's CBC solver:

```python
solver = PULP_CBC_CMD(msg=True, timeLimit=1800, gapRel=0.01)
model.solve(solver)
```

The solver output provided the following results:

```
Result - Optimal solution found

Objective value:                19442586.70295466
Enumerated nodes:               0
Total iterations:               130
Time (CPU seconds):             0.09
Time (Wallclock seconds):       0.09
```

This output indicates that:
1. An optimal solution was found, representing the best possible fleet composition given our constraints.
2. The total cost over the 16-year period is approximately $19.4 million.
3. The solution was found efficiently (in 0.09 seconds) without requiring branching in the branch-and-bound algorithm.

The optimal solution provides a year-by-year plan for vehicle purchases, usage, and sales that minimizes total cost while meeting demand and emission constraints. This plan can serve as a roadmap for fleet managers to guide their decarbonization efforts.

Looking back, I'm satisfied with the approach I took, but I'm also curious about other potential methods:
- Could a different objective function, perhaps one that more explicitly balances cost and emissions, yield interesting insights?
- How would the results change if we incorporated uncertainty in fuel prices or demand?
- Might there be heuristic approaches that could find good (if not optimal) solutions more quickly for even larger problem instances?

 I'm eager to learn more about how others approached this problem.