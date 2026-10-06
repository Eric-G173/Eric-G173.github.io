## Enhancement Three: 
### Enhanced Code:
**Python:**
```
import os
from flask import Flask, jsonify
from flask_cors import CORS
from dotenv import load_dotenv
import mysql.connector

load_dotenv()

app = Flask(__name__)
CORS(app)  # allows the React dev server (port 5173) to call this API


def get_connection():
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME"),
    )

# Gets the number of items that were returned.
@app.get("/api/return-counts")
def return_counts():
    try:
        conn = get_connection()
        cursor = conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT c.State AS state,
                   o.SKU AS sku,
                   o.Description AS description,
                   COUNT(*) AS returns
            FROM RMA r
            JOIN Orders o ON r.OrderID = o.OrderID
            JOIN Collaborators c ON o.CollaboratorID = c.CollaboratorID
            GROUP BY c.State, o.SKU, o.Description
        """)
        rows = cursor.fetchall()
        cursor.close()
        conn.close()
        return jsonify(rows)
    except mysql.connector.Error as err: #General database error catch
        print(err)
        return jsonify({"error": "Database error"}), 500


# Must run app.py in order for the app to run
if __name__ == "__main__":
    app.run(port=3001, debug=True)
```

React
{% raw %}
```jsx
import { useEffect, useState } from 'react';
import './App.css';

const API_URL = 'http://localhost:3001/api/return-counts';

// Turn the API rows into lookup-friendly data
function buildData(rows) {
  const counts = {}; // counts[state][sku] = returns
  const descriptions = {}; // descriptions[sku] = product description
  for (const row of rows) {
    if (!counts[row.state]) counts[row.state] = {};
    counts[row.state][row.sku] = row.returns;
    descriptions[row.sku] = row.description;
  }
  const states = Object.keys(counts).sort();
  const products = Object.keys(descriptions)
    .sort()
    .map((sku) => ({ sku, description: descriptions[sku] }));
  return { states, products, counts };
}

// Returns for a state/product combination. A null filter means all states
// A state/product pair with no rows means zero returns.
function countReturns(data, state, sku) {
  const states = state ? [state] : data.states;
  const skus = sku ? [sku] : data.products.map((p) => p.sku);
  let total = 0;
  for (const s of states) {
    for (const k of skus) total += data.counts[s]?.[k] ?? 0;
  }
  return total;
}

// Reusable container
function Panel({ title, className = '', children }) {
  return (
    <section className={`panel ${className}`}>
      <header className="panel-header">{title}</header>
      <div className="panel-body">{children}</div>
    </section>
  );
}

// Row of buttons for choosing a sort order
function SortToggle({ options, value, onChange, label }) {
  return (
    <div className="sort-toggle" role="group" aria-label={label}>
      {options.map(([optionValue, text]) => (
        <button
          key={optionValue}
          type="button"
          className={value === optionValue ? 'active' : ''}
          aria-pressed={value === optionValue}
          onClick={() => onChange(optionValue)}
        >
          {text}
        </button>
      ))}
    </div>
  );
}

// Search bar on top, then an optional toolbar, then a list.
// The all row is default
// Click an item to select it, or click it again to go back to all
function SearchList({
  items,
  getKey,
  getText,
  getMeta,
  selectedKey,
  onSelect,
  placeholder,
  allLabel,
  allMeta,
  toolbar,
}) {
  const [query, setQuery] = useState('');

  const matches = items.filter((item) =>
    getText(item).toLowerCase().includes(query.trim().toLowerCase())
  );

  return (
    <div className="search-list">
      <input
        type="search"
        className="search-input"
        placeholder={placeholder}
        aria-label={placeholder}
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      {toolbar}
      <ul className="list">
        <li className="list-all-row">
          <button
            type="button"
            className={`list-item list-all ${selectedKey === null ? 'selected' : ''}`}
            aria-pressed={selectedKey === null}
            onClick={() => onSelect(null)}
          >
            <span className="list-text">{allLabel}</span>
            <span className="list-meta">{allMeta}</span>
          </button>
        </li>
        {matches.map((item) => {
          const key = getKey(item);
          const isSelected = key === selectedKey;
          return (
            <li key={key}>
              <button
                type="button"
                className={`list-item ${isSelected ? 'selected' : ''}`}
                aria-pressed={isSelected}
                onClick={() => onSelect(isSelected ? null : key)}
              >
                <span className="list-text">{getText(item)}</span>
                <span className="list-meta">{getMeta(item)}</span>
              </button>
            </li>
          );
        })}
        {matches.length === 0 && <li className="list-empty">No matches found</li>}
      </ul>
    </div>
  );
}

function App() {
  const [data, setData] = useState(null);
  const [error, setError] = useState('');
  const [selectedState, setSelectedState] = useState(null);
  const [selectedSku, setSelectedSku] = useState(null);
  const [stateSort, setStateSort] = useState('highest'); // 'highest', 'lowest' or 'alpha'
  const [sortOrder, setSortOrder] = useState('highest'); // 'highest' or 'lowest'

  // Load the return counts from the Flask API (which reads MySQL)
  useEffect(() => {
    fetch(API_URL)
      .then((res) => {
        if (!res.ok) throw new Error(`Server returned ${res.status}`);
        return res.json();
      })
      .then((rows) => setData(buildData(rows)))
      .catch((err) => setError(err.message));
  }, []);

  if (error) {
    return (
      <main className="app">
        <p className="status error">
          Could not load data ({error}). Check that Flask is running on port 3001 and
          MySQL is up.
        </p>
      </main>
    );
  }

  if (!data) {
    return (
      <main className="app">
        <p className="status">Loading return data...</p>
      </main>
    );
  }

  const product = data.products.find((p) => p.sku === selectedSku);

  // Every state with its returns (for the selected product, or all products)
  const stateRows = data.states
    .map((name) => ({ name, returns: countReturns(data, name, selectedSku) }))
    .sort((a, b) => {
      if (stateSort === 'highest') return b.returns - a.returns;
      if (stateSort === 'lowest') return a.returns - b.returns;
      return a.name.localeCompare(b.name);
    });

  // Every product with its returns (for the selected state, or all states)
  const productRows = data.products.map((p) => ({
    ...p,
    returns: countReturns(data, selectedState, p.sku),
  }));

  // Same rows sorted for the product return counts panel
  const productCounts = [...productRows].sort((a, b) =>
    sortOrder === 'highest' ? b.returns - a.returns : a.returns - b.returns
  );
  const maxReturns = Math.max(...productCounts.map((p) => p.returns), 1);

  // Main return function, all displayable information is here
  return (
    <main className="app">
      <div className="top-row">
        <Panel title="State filter" className="states-panel">
          <SearchList
            items={stateRows}
            getKey={(row) => row.name}
            getText={(row) => row.name}
            getMeta={(row) => row.returns.toLocaleString()}
            selectedKey={selectedState}
            onSelect={setSelectedState}
            placeholder="Search states"
            allLabel="All states"
            allMeta={countReturns(data, null, selectedSku).toLocaleString()}
            toolbar={
              <SortToggle
                label="Sort states"
                value={stateSort}
                onChange={setStateSort}
                options={[
                  ['highest', 'Most returns'],
                  ['lowest', 'Fewest returns'],
                  ['alpha', 'A to Z'],
                ]}
              />
            }
          />
        </Panel>

        <Panel title="Filter product return" className="results-panel">
          <SearchList
            items={productRows}
            getKey={(p) => p.sku}
            getText={(p) => `${p.sku} - ${p.description}`}
            getMeta={(p) => p.returns.toLocaleString()}
            selectedKey={selectedSku}
            onSelect={setSelectedSku}
            placeholder="Search products"
            allLabel="All products"
            allMeta={countReturns(data, selectedState, null).toLocaleString()}
          />
        </Panel>
      </div>

      <section className="results-bar" aria-live="polite">
        <div className="result-cell">
          <span className="result-label">State</span>
          <span className="result-value">{selectedState || 'All states'}</span>
        </div>
        <div className="result-cell">
          <span className="result-label">Product</span>
          <span className="result-value">
            {product ? `${product.sku} - ${product.description}` : 'All products'}
          </span>
        </div>
        <div className="result-cell result-total">
          <span className="result-label">Returns</span>
          <span className="result-value">
            {countReturns(data, selectedState, selectedSku).toLocaleString()}
          </span>
        </div>
      </section>

      <Panel
        title={`Product return counts (${selectedState || 'All states'})`}
        className="counts-panel"
      >
        <SortToggle
          label="Sort products by returns"
          value={sortOrder}
          onChange={setSortOrder}
          options={[
            ['highest', 'Highest first'],
            ['lowest', 'Lowest first'],
          ]}
        />

        <ol className="count-list">
          {productCounts.map((p) => (
            <li key={p.sku} className="count-row">
              <span className="count-name">
                {p.sku} - {p.description}
              </span>
              <span className="count-bar">
                <span
                  className="count-fill"
                  style={{ width: `${(p.returns / maxReturns) * 100}%` }}
                />
              </span>
              <span className="count-number">{p.returns.toLocaleString()}</span>
            </li>
          ))}
        </ol>
      </Panel>
    </main>
  );
}

export default App;
```
{% endraw %}

CSS
```
[Uploadi:root {
  --ink: #1b1f24;
  --paper: #ffffff;
  --muted: #5b6570;
  --line: #1b1f24;
  --header-bg: #eef1f4;
  --accent: #2f6fed;
  --gap: 24px;
}
{% endraw %}
.app {
  max-width: 1100px;
  margin: 0 auto;
  padding: var(--gap);
  display: flex;
  flex-direction: column;
  gap: var(--gap);
}

/* Shared container style */
.panel {
  display: flex;
  flex-direction: column;
  border: 2px solid var(--line);
  background: var(--paper);
  min-width: 0;
}

.panel-header {
  padding: 8px 14px;
  border-bottom: 2px solid var(--line);
  background: var(--header-bg);
  font-weight: 600;
}

.panel-body {
  flex: 1;
  min-height: 0;
  padding: 14px;
}

/* Top row: wide states panel, narrower products panel */
.top-row {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: var(--gap);
}

.states-panel,
.results-panel {
  height: 340px;
}

/* Search bar + scrolling list */
.search-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
  height: 100%;
}

.search-input {
  width: 100%;
  box-sizing: border-box;
  padding: 8px 10px;
  border: 2px solid var(--line);
  font: inherit;
  background: var(--paper);
}

.search-input:focus-visible,
.list-item:focus-visible,
.sort-toggle button:focus-visible {
  outline: 3px solid var(--accent);
  outline-offset: 1px;
}

.list {
  flex: 1;
  min-height: 0;
  margin: 0;
  padding: 0;
  list-style: none;
  overflow-y: auto;
  border: 1px solid #c9d0d8;
}

.list-item {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  width: 100%;
  padding: 7px 10px;
  border: none;
  border-bottom: 1px solid #e3e7ec;
  background: transparent;
  font: inherit;
  text-align: left;
  cursor: pointer;
}

.list-item:hover {
  background: var(--header-bg);
}

.list-item.selected {
  background: var(--ink);
  color: var(--paper);
}

.list-meta {
  font-variant-numeric: tabular-nums;
  font-weight: 600;
}

/* "All" row stays visible at the top while the list scrolls */
.list-all-row {
  position: sticky;
  top: 0;
  z-index: 1;
  background: var(--header-bg);
  border-bottom: 2px solid var(--line);
}

.list-all {
  font-weight: 600;
}

.search-list .sort-toggle {
  align-self: flex-start;
  margin-bottom: 0;
}

.list-empty {
  padding: 7px 10px;
  color: var(--muted);
}

/* Results bar */
.results-bar {
  display: grid;
  grid-template-columns: 1fr 2fr auto;
  border: 2px solid var(--line);
  background: var(--paper);
}

.result-cell {
  display: flex;
  flex-direction: column;
  gap: 2px;
  padding: 10px 16px;
  border-right: 2px solid var(--line);
  min-width: 0;
}

.result-cell:last-child {
  border-right: none;
}

.result-label {
  font-size: 0.85rem;
  color: var(--muted);
}

.result-value {
  font-weight: 600;
  overflow-wrap: anywhere;
}

.result-total {
  min-width: 140px;
  background: var(--header-bg);
}

.result-total .result-value {
  font-size: 1.6rem;
}

/* Product return counts */
.sort-toggle {
  display: inline-flex;
  margin-bottom: 14px;
  border: 2px solid var(--line);
}

.sort-toggle button {
  padding: 6px 14px;
  border: none;
  background: var(--paper);
  font: inherit;
  cursor: pointer;
}

.sort-toggle button + button {
  border-left: 2px solid var(--line);
}

.sort-toggle button.active {
  background: var(--ink);
  color: var(--paper);
}

.count-list {
  margin: 0;
  padding: 0;
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.count-row {
  display: grid;
  grid-template-columns: minmax(180px, 1fr) 2fr 70px;
  align-items: center;
  gap: 12px;
}

.count-bar {
  height: 14px;
  background: var(--header-bg);
}

.count-fill {
  display: block;
  height: 100%;
  background: var(--accent);
}

.count-number {
  text-align: right;
  font-variant-numeric: tabular-nums;
  font-weight: 600;
}

/* Narrow screens */
@media (max-width: 800px) {
  .top-row {
    grid-template-columns: 1fr;
  }
  .results-bar {
    grid-template-columns: 1fr;
  }
  .result-cell {
    border-right: none;
    border-bottom: 2px solid var(--line);
  }
  .result-cell:last-child {
    border-bottom: none;
  }
  .count-row {
    grid-template-columns: 1fr 70px;
  }
  .count-name {
    grid-column: 1 / -1;
  }
  .count-bar {
    grid-column: 1;
  }
}ng App.css…]()
```
### Original Document:
[DAD 220 Project Two.docx](https://github.com/user-attachments/files/33124512/DAD.220.Project.Two.docx)



### Explanation:
**What is the Artifact**
<br>
	This artifact is my final project of my DAD 220 class (Intro to Structured Databases). This was in the form of a google document that describes getting different pieces of information from the database, and writing a business proposal that incorporates all the data. The database itself is a mySQL database that contains information about different items that a company sells and the return rates by states. Throughout the document I was able to pull different lists of information such as percentage of item returns, the top 10 most returned items, etc. This project was created around August of 2026.
<br>
**Why I Selected the Artifact and What Improved**
<br>
	The reason I selected this artifact for the database portion is because the current main functionality behind this project involves the mySQL database. The main purpose of this artifact was to prepare a business proposal given certain SQL queries and explain them to a non-technical user. I believe this showcases an important part of why databases are here, in how they can store thousands of data pieces and you can extract and talk about them instantly. The main downside to a database is that you have to know what queries to run to fetch the data. That downside is what I wanted to target with this artifact, and I did so by proposing a front-end application to go with the database that way non-technical users have control of the information they gather. I believe that the complex queries being used on the backend will showcase my skills with the database, and the frontend will show my design and software design skills. The artifact will be improved by having a SPA application attached to the database, that will allow for data to be visualized more clearly, and allow for non-technical users to access data on their own.
<br>
**Meeting Course Outcomes**
<br>
	I think overall I did meet the outcomes for this enhancement. I was able to successfully implement the SQL queries into a front-end dashboard that is easier for non-technical users to read. My original goal was to further improve the idea of this project, which is to simplify the gathering of data to a usable webpage, and I think the final enhancement shows the different systems I implemented in order to do this. You can see through the different components that I had it match the original queries I was pulling for the project. I do not have any updates yet on my outcome coverage plan. I made sure to keep in line with the original goals I set, and I believe I succeeded in implementing them.
<br>
**What I Learned**
<br>
	Overall, I learned a lot about how database connections work and how you can convert SQL queries into user interfaces that can be interacted with. I already had slight experience with this in the first artifact, but this one had me fully build the backend fetching from scratch, as well as the front-end UI. I think using React was helpful for the front-end because it gave a really responsive SPA that is fast and easy to manage. This also helped develop my backend skills even further, which is really crucial for my future career as a software engineer. 
