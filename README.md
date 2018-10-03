# SQL-Server-Context-Provider

## Prerequisites
- SQL Server 2012
- Visual Studio (any version that supports .Net Core 2.1)
- [XMPro IoT Framework NuGet package](https://www.nuget.org/packages/XMPro.IOT.Framework/3.0.2-beta)
- Please see the [Building an Agent for XMPro IoT](https://docs.xmpro.com/lessons/writing-an-agent-for-xmpro-iot/) guide for a better understanding of how the XMPro IoT Framework works

## Description
The *SQL Server Context Provider* allows the user to add context to a stream by bringing in reference/static data from a configured SQL data source. This agent is usually used in conjunction with the *Join Transformation*, which allows two data sources to be joined together to form a new data source for the stream.

## How the code works
All settings referred to in the code need to correspond with the settings defined in the template that has been created for the agent using the Stream Integration Manager. Refer to the [Stream Integration Manager](https://docs.xmpro.com/courses/packaging-an-agent-using-stream-integration-manager/) guide for instructions on how to define the settings in the template and package the agent after building the code. 

After packaging the agent, you can upload it to XMPro IoT and start using it.

### Settings
When a user needs to use the *SQL Server Context Provider* agent, they need to provide the name of the SQL Server instance they want to connect to, along with a username and password that can be used. Retrieve these values from the configuration using the following code: 

```csharp
private Configuration config;
private SqlConnection connection;
private string SQLServer => this.config["SQLServer"];
private string SQLUser => this.config["SQLUser"];
private string SQLPassword => this.decrypt(this.config["SQLPassword"]);
```

Next, the user will have to specify the name of the database and table the agent needs to get the data from. Also, the user needs to specify if SQL Authentication or Windows Authentication should be used. In the template, this is done by checking the *Use SQL Authentication* check box.

```csharp
private bool SQLUseSQLAuth
{
    get
    {
        var temp = false;
        bool.TryParse(this.config["SQLUseSQLAuth"], out temp);
        return temp;
    }
}

private string SQLDatabase => this.config["SQLDatabase"];
private string SQLTable => this.config["SQLTable"];
```
### Configurations
In the *GetConfigurationTemplate* method, parse the JSON representation of the settings into the Settings object.
```csharp
var settings = Settings.Parse(template);
new Populator(parameters).Populate(settings);
```
Next, create the correct control for each setting and set its value:
```csharp
TextBox SQLServer = settings.Find("SQLServer") as TextBox;
SQLServer.HelpText = string.Empty;
TextBox SQLUser = settings.Find("SQLUser") as TextBox;
CheckBox SQLUseSQLAuth = settings.Find("SQLUseSQLAuth") as CheckBox;
TextBox SQLPassword = settings.Find("SQLPassword") as TextBox;
SQLPassword.Visible = SQLUseSQLAuth.Value;
```
Each database needs to be listed; thus, letting the user choose a database to connect to. To do this, get all the tables available to choose from, using the SQL Server instance name, username and password as provided by the user.
```csharp
IList<string> tables = SQLHelpers.GetTables(SQLServer, SQLUser, SQLUseSQLAuth, this.decrypt(SQLPassword.Value), SQLDatabase, out            errorMessage);
```

Create a drop down to list the tables in.
```csharp
DropDown SQLTable = settings.Find("SQLTable") as DropDown;
SQLTable.Options = tables.Select(i => new Option() { DisplayMemeber = i, ValueMemeber = i }).ToList();
```

### Validate
For this agent to be successfully added, the following needs to be true:
* The SQL Server instance name should have a value.
* The username should have a value.
* If SQL Server Authentication is used, a password needs to be specified.
* The database and table name needs to be specified.

```csharp
int i = 1;
var errors = new List<string>();
this.config = new Configuration() { Parameters = parameters };

if (String.IsNullOrWhiteSpace(this.SQLServer))
    errors.Add($"Error {i++}: SQL Server is not specified.");

if (String.IsNullOrWhiteSpace(this.SQLUser))
    errors.Add($"Error {i++}: Username is not specified.");

if (this.SQLUseSQLAuth && String.IsNullOrWhiteSpace(this.SQLPassword))
    errors.Add($"Error {i++}: Password is not specified.");

if (String.IsNullOrWhiteSpace(this.SQLDatabase))
    errors.Add($"Error {i++}: Database is not specified.");

if (String.IsNullOrWhiteSpace(this.SQLTable))
    errors.Add($"Error {i++}: Table is not specified.");
```

If all of the values are provided, make sure that the table exists.

```csharp
var server = new TextBox() { Value = this.SQLServer };
var errorMessage = "";

IList<string> tables = SQLHelpers.GetTables(server, new TextBox() { Value = this.SQLUser }, new CheckBox() { Value =                        this.SQLUseSQLAuth }, this.SQLPassword, new DropDown() { Value = this.SQLDatabase }, out errorMessage);

if (string.IsNullOrWhiteSpace(errorMessage) == false)
{
    errors.Add($"Error {i++}: {errorMessage}");
    return errors.ToArray();
}

if (tables.Any(d => d == this.SQLTable) == false)
    errors.Add($"Error {i++}: Table '{this.SQLTable}' cannot be found in {this.SQLDatabase}.");
```

### Create
Set the config variable to the configuration received in the *Create* method.
```csharp
public void Create(Configuration configuration)
{
    this.config = configuration;
}
```
### Start
Build a connection string and create a connection to the SQL Server instance.
```csharp
public void Start()
{
    this.connection = new SqlConnection(SQLHelpers.GetConnectionString(SQLServer, SQLUser, SQLPassword, SQLUseSQLAuth, SQLDatabase));
}
```

### Destroy
Make sure you close the connection to the SQL Server instance.
```csharp
public void Destroy()
{
    try
    {
        connection.Close();
    }
    catch
    {
        connection?.Dispose();
    }
}
```

### Publishing Events
Publish the events by implementing the *Poll* method and invoking the *OnPublish* event.
```csharp
public void Poll()
{
    using (SqlDataAdapter a = new SqlDataAdapter(string.Format("SELECT * FROM {0}", SQLHelpers.AddTableQuotes(SQLTable)), connection))
    {
        DataTable dt = new DataTable();
        a.Fill(dt);
        IList<IDictionary<string, object>> rtr = new List<IDictionary<string, object>>();
        foreach (DataRow row in dt.Rows)
        {
            IDictionary<string, object> r = new Dictionary<string, object>();
            foreach (DataColumn col in dt.Columns)
                r.Add(col.ColumnName, row[col]);
            rtr.Add(r);
        }
        if (rtr.Count > 0)
            this.OnPublish?.Invoke(this, new OnPublishArgs(rtr.ToArray()));
    }
}
```
### Decrypting Values
Since this agent needs secure settings (*SQL Password*), the value will automatically be encrypted. Use the following code to decrypt the value.
```csharp
var request = new OnDecryptRequestArgs(value);
this.OnDecryptRequest?.Invoke(this, request);
return request.DecryptedValue;
```
