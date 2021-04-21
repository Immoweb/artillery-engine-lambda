# Artillery.io SQL Plugin

<p align="center">
    <em>Load test SQL with <a href="https://artillery.io">Artillery.io</a></em>
</p>

Based on the [Lambda Engine by Orchestrated.io](https://github.com/orchestrated-io/artillery-engine-lambda).

## Usage

**Important:**

- The plugin requires Artillery `1.5.8-3` or higher.
- This plugin uses [AnyDB](https://github.com/grncdr/node-any-db) as an underlying method to abstract the DB connectors. You need to install the appropiate connector to your database.

### Install the plugin

```
# If Artillery is installed globally:
npm install -g artillery-engine-sql
```

### Use the plugin

1. Set the `engine` property of the scenario to `sql`.
2. Set the target to either a string or an object (it's passed directly to AnyDB) as indicated [here](https://github.com/grncdr/node-any-db#api).
3. Use `query` in your scenario to execute the SQL statement. You can pass either a string or an object.

#### Query as String

You can pass either a SQL Query string to read from. The artillery template will be applied upon it, so statements like 'SELECT * from {{ var }}' are supported.

#### Query as an Object

When stating the query as an object the parameter 'statement' is required and all the rest are optional. The parameter 'statement' has support for files and also for template statements.
The interfaces for 'beforeRequest' and 'afterResponse' are the same as in the [HTTP engine](https://artillery.io/docs/http-reference/#advanced-writing-custom-logic-in-javascript), where 'requestParams' is the target configuration and the query itself. You can modify the query dynamically in beforeRequest before it's executed.

```yaml
- query:
  -
    statement: 'SELECT * from test where id = ?'
    values:
      - 45
    beforeRequest: "before_fn"
    afterResponse: "after_fn"
```

#### Example Script

```yaml
config:
  target: "driver://user:pass@hostname/database"
  phases:
    - arrivalCount: 10
      duration: 1
  engines:
    sql: {}

scenarios:
  - name: "SQL query"
    engine: "sql"
    flow:
      - loop:
        - query: 'SELECT * from test'
        - think: 1
        count: 100
```

(See [example.yml](example.yml) for a complete example.)

#### Example Script with a long SQL Statement

You can use the above approach to read long SQL statements from a file, see the following example snippet:

```yaml
scenarios:
  - name: "DB Query"
    engine: "sql"
    flow:
      - query:
          beforeRequest: "loadQuery"
          statement: "./query.sql"
```

In 'query.sql' you would have the SQL statement, as follows:

```sql
Select * from test
```

In your processor file, you would have something like this:

```typescript
interface UserContext {
    vars: { [k: string]: string | number };
}

interface DBParams {
    query: string;
    args: string[];
    afterResponse: Function;
    beforeRequest: Function;
    target: Object;
}

function loadQuery(params: DBParams, context: UserContext, ee: EventListener, next: (err?: Error) => void) {
    let filePath = params.query;
    if (filePath !== path.basename(filePath)) {
        // the query is a path
        if (!path.isAbsolute(filePath) && !fs.existsSync(filePath)) {
            // it may be relative to us
            filePath = path.resolve(__dirname, filePath);
        }
        if (fs.existsSync(filePath) && fs.lstatSync(filePath).isFile()) {
            params.query = fs.readFileSync(filePath).toString();
        }
    }
    next();
}
module.exports = { loadQuery: loadQuery };
```

### Run Your Script

```bash
artillery run my_script.yml
```

### License

[MPL 2.0](https://www.mozilla.org/en-US/MPL/2.0/)
