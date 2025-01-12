## 1
```c#
using DotNetEnv;
using Azure.AI.OpenAI;
using Microsoft.Extensions.AI;
using Azure;
using Npgsql;
using System.Text.Json;
using System.Threading.Tasks;

public class Program
{
    private static readonly string _tableSchema = @"
Table: mytransaction
Columns:
- transactiondate (date)
- portfoliocode (varchar)
- portfolioname (varchar)
- familyname (varchar)
- foliono (varchar)
- rtaschemecode (varchar)
- schemename (varchar)
- units (numeric)
- transactiontype (varchar)
- transactionamount (numeric)
- costperunit (numeric)
- load (numeric)
- stt (numeric)
- stampduty (numeric)
- product (varchar)
- rmname (varchar)
- description (text)
- vector (double precision[])
";

    private static readonly string _connectionString = "Host=localhost;Database=wealthai;Username=postgres;Password=Kgisl@12345";

    static async Task Main(string[] args)
    {
        Env.Load(".env");
        string githubKey = Env.GetString("GITHUB_KEY");

        // Add the chat client
        IChatClient client =
            new AzureOpenAIClient(
                new Uri("https://models.inference.ai.azure.com"),
                    new AzureKeyCredential(githubKey))
                    .AsChatClient(modelId: "gpt-4o-mini");

        // Test the chatWithDatabase function
        var testPrompt = "give me the rmname and no of portfolionames they managed in the mytransaction table.";
        var response = await ChatWithDatabase(client, testPrompt);
        Console.WriteLine(response);


        // var response = await client.CompleteAsync("What is AI?");

        // Console.WriteLine(response.Message);
    }

    static async Task<string> ChatWithDatabase(IChatClient client, string prompt)
    {
        // Use the model to generate a SQL query
        var response = await client.CompleteAsync($"{_tableSchema}\n{prompt}");

        // Extract the SQL query from the response
        var sqlQuery = ExtractSqlQuery(response.Message.Text);
        Console.WriteLine("Query:"+sqlQuery);

        // Execute the SQL query and return the result
        if (sqlQuery.StartsWith("SELECT", StringComparison.OrdinalIgnoreCase))
        {
            var result = await ExecuteQuery(sqlQuery);
            return $"Query result: {JsonSerializer.Serialize(result)}";
        }
        else
        {
            return "Only SELECT queries are supported.";
        }
    }

    static string ExtractSqlQuery(string response)
    {
        var startIndex = response.IndexOf("```sql", StringComparison.OrdinalIgnoreCase);
        if (startIndex == -1) return "";

        startIndex += 7; // Move past "```sql"
        var endIndex = response.IndexOf("```", startIndex, StringComparison.OrdinalIgnoreCase);
        if (endIndex == -1) return "";

        return response.Substring(startIndex, endIndex - startIndex).Trim();
    }

    static async Task<NpgsqlDataReader> ExecuteQuery(string query)
    {
        using var connection = new NpgsqlConnection(_connectionString);
        await connection.OpenAsync();

        using var command = new NpgsqlCommand(query, connection);
        return await command.ExecuteReaderAsync();
    }
}

```

