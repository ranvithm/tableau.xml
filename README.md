# Tableau .twb XML Structure

A Tableau workbook file (.twb) is actually an XML document describing all the workbook’s metadata (data connections, fields, sheets, etc.) – not the data itself. Understanding its structure lets developers parse and inspect workbooks programmatically. Below is a detailed guide to the key XML elements and attributes, with examples.

## Workbook Root Element

The root of a .twb XML file is the `<workbook>` tag. Its attributes identify the Tableau version and platform used to build the workbook:
```xml
<?xml version='1.0' encoding='utf-8' ?>
<!-- build 20223.22.1005.1835 -->
<workbook locale='en_GB'
          source-build='2022.3.0 (20223.22.1005.1835)'
          source-platform='win'
          version='18.1'
          xmlns:user='http://www.tableausoftware.com/xml/user'>
  ...
</workbook>
```
- **source-build** and **version**: Indicate the Tableau version that created the workbook. For example, `source-build='2022.3.0 (...)' version='18.1'`.
- **source-platform**: Indicates OS ('win' or 'mac').
- Other attributes: May include locale (user locale), original-version (if saved down), and XML namespaces.
- All workbook-wide settings (shelf defaults, color palettes, etc.) often appear inside a `<preferences>` element immediately under `<workbook>`.

## Data Sources Section

Within `<workbook>`, the `<datasources>` element lists all embedded data sources. Each `<datasource>` entry describes connection details, tables/relations, and columns:
```xml
<datasources>
  <datasource name='MyData' inline='true' version='2023.1'>
    <connection class='excel-direct' filename='SampleData.xlsx' />
    <relation name='Orders' table='[Orders$]' type='table' />
    <metadata-records>
      <metadata-record class='column'>
        <remote-name>Order ID</remote-name>
        <remote-type>130</remote-type>
        <local-name>[Order ID]</local-name>
        <local-type>string</local-type>
      </metadata-record>
      <!-- ...more columns -->
    </metadata-records>
  </datasource>
  <!-- Other datasources... -->
</datasources>
```
Key points:
- `<datasource>`: Defines each data source. Attributes include name, version, and flags like inline (embedded vs. published) or hasconnection. Parameters are also stored as a fake “Parameters” datasource.
- `<connection>`: Specifies connection type (e.g. `class='excel-direct'`, `class='sqlserver'`, etc.), access mode (readonly/readwrite), file or server name, and more.
- `<relation>`: Lists each table or custom SQL. It has attributes:
  - `type='table'` for tables or `type='text'` for custom SQL (in which case the SQL query appears as text content).
  - `name` and `table` (table name in the database or file) when `type='table'`.
- `<columns>` (inside `<relation>`): Lists raw column definitions from the source table (name, data type, ordinal).
- `<metadata-records>`: Contains `<metadata-record>` elements, one per field. These provide the field’s data type, alias, aggregation, and other metadata. They represent Tableau’s view of the field (for example, local name in Tableau versus remote name in source).

## Field-Level Metadata

Individual fields (columns) in a data source appear in two places:
- In `<relation>` under `<columns>` for raw source columns.
- In `<datasource>` (or “Parameters” datasource) as `<column>` entries describing Tableau fields (including parameters, calculated fields, and bins).
- In `<metadata-records>` listing metadata about each column (aggregation, null count, etc.).

For example, a parameter stored as a field looks like:
```xml
<column caption='Top Customers' datatype='integer' name='[Parameter 1]'
        param-domain-type='range' role='measure' type='quantitative' value='5'>
  <calculation class='tableau' formula='5'/>            <!-- default value -->
  <range granularity='5' min='5' max='20'/>
</column>
```
And a calculated field looks like:
```xml
<column caption='Profit Ratio' datatype='real' default-format='p0%'
        name='[Calculation_5571209093911105]' role='measure' type='quantitative'>
  <calculation class='tableau' formula='SUM([Profit])/SUM([Sales])'
               scope-isolation='false'/>
</column>
```
Each `<column>` element under a datasource or metadata context can include:
- `caption`, `datatype`, `role` (dimension/measure), etc. (Descriptive attributes).
- A child `<calculation>` element if it’s a calculated field or parameter (with the formula).
- For parameters, `<range>` or `<value>` lists for the domain (continuous range or discrete list).
- In `<metadata-record>`, tags like `<remote-name>`, `<local-name>`, `<local-type>`, `<aggregation>`, etc., capture statistics and formatting.

## Parameters & Calculated Fields

Parameters are represented as an inline datasource named "Parameters". Each parameter is a `<column>` with `param-domain-type` and default value. Its default value is shown via a `<calculation>` child, and its allowable values via `<range>` or `<value>` children. Calculated fields appear under their respective datasource as `<column>` elements named like `[Calculation_xxxxx]`, with a child `<calculation>` holding the formula. The `class='tableau'` attribute indicates a Tableau calculation.

## Worksheets Section

The `<worksheets>` section contains one `<worksheet>` element per sheet. Each worksheet element encodes what fields are on each shelf and filter. For example:
```xml
<worksheets>
  <worksheet name='Sales by Category'>
    <table>
      <view>
        <datasources>
          <datasource name='Sample - Superstore'/>
          <datasource name='Parameters'/>
        </datasources>
        <datasource-dependencies datasource='Sample - Superstore'>
          <column datatype='string' name='[Category]' role='dimension' type='nominal'/>
          <column datatype='real' name='[Sales]' role='measure' type='quantitative'/>
          <!-- other fields used -->
        </datasource-dependencies>
        <datasource-dependencies datasource='Parameters'>
          <column caption='Top Customers' datatype='integer' name='[Parameter 1]' ...>
            <calculation class='tableau' formula='5'/>
            <!-- parameter default/range -->
          </column>
        </datasource-dependencies>
        <filter class='categorical' column='[Sample - Superstore].[Segment]'/>
        <!-- more filter elements -->
        <rows>[Sample - Superstore].[none:Category:nk]</rows>    <!-- Row shelf -->
        <cols>[Sample - Superstore].[sum:Sales:qk]</cols>        <!-- Column shelf -->
      </view>
      <!-- style, panes, etc. -->
    </table>
  </worksheet>
  <!-- more worksheets -->
</worksheets>
```
In detail:
- `<worksheet name=...>`: Defines a sheet by name.
- Inside `<view>`:
  - `<datasources>` and `<datasource-dependencies>` list which data sources (and which fields) are used on this sheet.
  - Within `<datasource-dependencies>`, each `<column>` element corresponds to a field on some shelf (rows, columns, detail, tooltip, etc.), with attributes like `role='dimension'` or `measure`.
  - `<filter>` entries: Each filter applied to the sheet. E.g. `<filter class='categorical' column='[...].[Field]'>`. (Filters can be of various types: categorical, range, etc.)
  - `<rows>` and `<cols>`: Explicitly list what is on the Rows and Columns shelves.
  - `<style>`: Formatting settings (fonts, colors, tooltip content, etc.; often complex XML).
  - `<panes>`, `<mark>`, `<encodings>`: Data panes and mark card encodings. For example, `<mark class='Automatic'/>` indicates the mark type (Automatic vs. e.g. Bar), and inside `<encodings>` you’ll see `<size>`, `<color>`, `<detail>`, `<tooltip>`, or `<lod>` elements linking to fields on those card slots.

Each worksheet’s XML therefore captures exactly which fields go where.

## Shelf Mappings and Field References

In the XML, fields are often referenced with fully-qualified names. For example:
```xml
<rows>[Sample - Superstore].[none:Category:nk]</rows>
```
This means the field `Category` (with Tableau-specific prefix/suffix `[none:Category:nk]`) from the `Sample - Superstore` source is on Rows. Similarly, encodings on the Marks card use syntax like:
```xml
<encodings>
  <size column='[Sample - Superstore].[sum:Profit:qk]'/>
  <lod column='[Sample - Superstore].[none:Country:nk]'/>
</encodings>
```
Each `<column>` or `<column-instance>` within `<datasource-dependencies>` refers to fields by `[datasource].[FieldName]`. These fully-qualified references ensure Tableau can match fields to their data sources.

## Dashboards Section

The `<dashboards>` element defines each dashboard layout. A `<dashboard>` entry includes nested `<zone>` elements that represent layout containers and placed objects. For example:
```xml
<dashboards>
  <dashboard name='Main Dashboard'>
    <style/>
    <size width='800' height='600'/>
    <zone h='100000' w='100000' x='0' y='0' type='layout-basic'>
      <zone name='Sales by Category' h='50000' w='80000' x='0' y='0' type='worksheet'/>
      <zone name='Profit Map'          h='50000' w='80000' x='0' y='50000' type='worksheet'/>
    </zone>
    <!-- possibly more zones: text boxes, images, or containers -->
  </dashboard>
  <!-- more dashboards -->
</dashboards>
```
Key points:
- Each `<dashboard>` has a `name` and may include `<size>` (fixed dimensions) and `<style>`.
- `<zone>` elements define layout. A zone with `type='layout-basic'` is a container. Zones with `type='worksheet'`, `type='text'`, etc., refer to placed objects.
- Zones have coordinates (`x`,`y`) and size (`w`,`h`) in a fixed-grid system (100000 units typically represents 100%). The attributes `name` (for worksheets) or `id` (for containers) identify the content.
- Nested zones represent container hierarchies (horizontal/vertical layouts).
- Actions (filter actions, URL actions, etc.) are stored under `<actions>` child elements within `<dashboard>`.

## Document-Level Metadata

Some workbook-wide metadata and settings are stored near the top:
- **Author/Description**: The `.twb` itself may not explicitly store author/email in XML. (File system metadata holds creator info.) Workbook title and description can be stored in elements like `<document-properties>` in some contexts.
- **`<preferences>`**: Holds UI defaults (shelf sizes, default formatting).
- **Color Palettes**: Custom palettes come from `Preferences.tps` file, but embedded/referenced palettes may appear as `<color-palette>` tags inside `<preferences>`.
- **Other metadata**: You may see `<document-format-change-manifest>` or legacy tags if upgrading older workbooks, mostly internal.

## Versioning and Compatibility

Tableau’s XML schema evolves with each release:
- The `<workbook>` tag’s `source-build` and `version` attributes help identify Tableau version used.
- New releases may introduce new XML elements or attributes; older elements may be deprecated.
- If downgraded, an `original-version` attribute may appear.
- Use forgiving XML parsers: ignore unknown tags rather than erroring.
- Scripts adapt by checking version attributes; minor schema changes usually safe if unrecognized tags are skipped.

## Parsing Tips (Python / Node.js)

When parsing `.twb` files:
- Use a proper XML parser (never regex). In Python, use `xml.etree.ElementTree` or `lxml.etree`. In Node.js, use `xml2js` or `fast-xml-parser`.
- Respect UTF-8 encoding (`<?xml version='1.0' encoding='utf-8'?>`).
- Escapes/Entities: Parsers normalize quotes/special chars (`'` vs. `&apos;`) without changing content.
- Namespaces: Handle tags with `xmlns:user` if present.
- Large files: Consider streaming (`iterparse`) if very large; most `.twb` are moderate (<10MB).
- Validation: After edits, open in Tableau to ensure validity.
- Security: Local files; low risk, but sanitize inputs if auto-generating XML.
- **Python example**:
  ```python
  import xml.etree.ElementTree as ET
  tree = ET.parse('workbook.twb')
  root = tree.getroot()
  for ds in root.findall('.//datasource'):
      print(ds.get('name'))
  ```
- **Node.js example**:
  ```js
  const { XMLParser } = require('fast-xml-parser');
  const fs = require('fs');
  const xmlData = fs.readFileSync('workbook.twb', 'utf-8');
  const parser = new XMLParser({ignoreAttributes: false});
  const obj = parser.parse(xmlData);
  console.log(obj.workbook.datasources);
  ```

## Example: Datasource Definition Breakdown

```xml
<datasource name="Sample - Superstore" inline="true" version="9.0">
  <!-- Connection details -->
  <connection class="excel-direct" filename="Sample - Superstore.xls" validate="no"/>
  <!-- Table (Orders$ from the Excel) -->
  <relation name="Orders$" table="[Orders$]" type="table">
    <!-- Source columns in Orders$ -->
    <columns header="yes" outcome="6">
      <column datatype="integer" name="Row ID" ordinal="0"/>
      <column datatype="string"  name="Order ID" ordinal="1"/>
      <!-- more columns... -->
    </columns>
  </relation>
  <!-- Field metadata -->
  <metadata-records>
    <metadata-record class="column">
      <remote-name>Order ID</remote-name>
      <remote-type>130</remote-type>
      <local-name>[Order ID]</local-name>
      <parent-name>[Orders$]</parent-name>
      <remote-alias>Order ID</remote-alias>
      <ordinal>1</ordinal>
      <local-type>string</local-type>
      <aggregation>Count</aggregation>
      <!-- other stats like contains-null, attributes -->
    </metadata-record>
    <!-- more metadata-record entries for each field -->
  </metadata-records>
</datasource>
```

Explanation:
- `<datasource name="Sample - Superstore" inline="true" version="9.0">`: Defines an inline datasource named “Sample - Superstore”.
- `<connection ...>`: Connection type and file reference.
- `<relation>`: Table definition or custom SQL.
- `<columns>`: Raw source columns.
- `<metadata-records>`: Tableau’s metadata per field: names, types, aggregation, etc.

This shows how the workbook XML encodes data source definitions.

---
