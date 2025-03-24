### Hi there, I'm Rahul 👋  
🔹 A Passionate Java Backend Developer  
🔹 Skilled in Java, JavaScript, and Web Technologies  
🔹 Open-source enthusiast 🚀 

![Rahul's GitHub Stats](https://github-readme-stats.vercel.app/api?username=Rahul1998sys&show_icons=true&theme=tokyonight)  

💬 Ask me about **Backend Development, Java & System Design**  
📫 Reach me at:  [LinkedIn] (https://www.linkedin.com/in/rahul-saxena-b25b811b2/) | [Twitter] (https://x.com/Saxen68846Rahul?t=FFWk-Soubr0Y7FbyJOButQ&s=09)  

**Rahul1998sys/Rahul1998sys** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on Web Based project
- 🌱 I’m currently learning React.js
- 👯 I’m looking to collaborate on ...
- 💬 Ask me about Anything
- 📫 How to reach me: LinkedIn, Twitter(X).

![GitHub followers](https://img.shields.io/github/followers/Rahul1998sys?style=social)  
![GitHub stars](https://img.shields.io/github/stars/Rahul1998sys?style=social)  
![Build Status](https://img.shields.io/github/workflow/status/Rahul1998sys/Chat-Server/CI)  

![Visitor Count](https://komarev.com/ghpvc/?username=Rahul1998sys&style=for-the-badge)
![Quote](https://github-readme-quotes.herokuapp.com/quote)

/***************************************************************************************************
 * Copyright 2012 (c) Daniel Schilling <http://stackoverflow.com/users/221708/daniel-schilling>
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy of this software and
 * associated documentation files (the "Software"), to deal in the Software without restriction,
 * including without limitation the rights to use, copy, modify, merge, publish, distribute,
 * sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in all copies or
 * substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT
 * NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
 * NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
 * DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
 **************************************************************************************************/

using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
using System.Linq;
using System.Text;
using Microsoft.SqlServer.Server;

namespace CodeGenerator
{
    /// <summary>
    /// A tool for generating unique random codes.
    /// </summary>
    /// <example>
    /// <code>
    /// using (var connection = new SqlConnection(ConnectionString))
    /// {
    ///     connection.Open();
    ///     var codeLength = GetCurrentCodeLength(connection);
    ///     using (var generator = new CodeGenerator(connection, codeLength))
    ///     {
    ///         var codes = generator.GenerateCodes(10000);
    ///         foreach(var code in codes)
    ///             Console.WriteLine(code);
    ///         if (generator.CodeLength > codeLength)
    ///         {
    ///             SaveNewCodeLength(generator.CodeLength);
    ///             NotifyDeveloperOfApproachingCodePoolExhaustion(
    ///                 generator.CodeLength,
    ///                 CodeGenerator.MaxCodeLength);
    ///         }
    ///     }
    /// }
    /// </code>
    /// </example>
    public class CodeGenerator : IDisposable
    {
        public const int MaxCodeLength = 8;
        private const string AvailableChars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";

        private const string Query = @"
            DECLARE @batchid uniqueidentifier;
            SET @batchid = NEWID();

            INSERT INTO dbo.Voucher (Code, BatchId)
            SELECT DISTINCT b.Code, @batchid
            FROM @batch b
            WHERE NOT EXISTS (
                SELECT Code
                FROM dbo.Voucher v
                WHERE b.Code = v.Code
            );

            SELECT Code
            FROM dbo.Voucher
            WHERE BatchId = @batchid;";

        private static readonly SqlMetaData[] BatchMetaData = new[]
        {
            new SqlMetaData("Code", SqlDbType.NVarChar, MaxCodeLength)
        };

        private readonly SqlConnection _connection;
        private readonly StringBuilder _builder;
        private readonly Random _random = new Random(Guid.NewGuid().GetHashCode());
        private readonly int _batchSize;
        private readonly double _collisionThreshold;
        private readonly SqlCommand _command;
        private readonly SqlParameter _batchParameter;

        private bool _disposed;

        public int CodeLength { get; private set; }

        /// <summary>
        /// Create a CodeGenerator instance.
        /// </summary>
        /// <param name="connection">
        /// The connection to the database.  Must be open.  Calling code is responsible for
        /// creating, opening, and disposing the connection.
        /// </param>
        /// <param name="codeLength">
        /// The initial code length, which will grow as needed as codes are used up.  However, you
        /// still need to persist the <c>CodeLength</c> property value and initialize this parameter
        /// correctly.  Otherwise, if you always supply the same initial
        /// <paramref name="codeLength"/> - say "4", then ALL of the 4-digit codes will eventually
        /// become used up instead of maintaining the sparseness dictated by the collision
        /// threshold.
        /// </param>
        public CodeGenerator(SqlConnection connection, int codeLength)
            : this(connection, codeLength, 500, 0.01)
        { }

        /// <summary>
        /// Create a CodeGenerator instance.
        /// </summary>
        /// <param name="connection">
        /// The connection to the database.  Must be open.  Calling code is responsible for
        /// creating, opening, and disposing the connection.
        /// </param>
        /// <param name="codeLength">
        /// The initial code length, which will grow as needed as codes are used up.  However, you
        /// still need to persist the <c>CodeLength</c> property value and initialize this parameter
        /// correctly.  Otherwise, if you always supply the same initial
        /// <paramref name="codeLength"/> - say "4", then ALL of the 4-digit codes will eventually
        /// become used up instead of maintaining the sparseness dictated by the
        /// <paramref name="collisionThreshold"/> parameter.
        /// </param>
        /// <param name="batchSize">
        /// The number of codes to generate, test, and insert at once.  Tune this value for best
        /// performance.  In my tests, 500 worked well.
        /// </param>
        /// <param name="collisionThreshold">
        /// A value between 0 (inclusive) and 1 (exclusive).  Supply a small value (perhaps 0.01) to
        /// keep codes sparse.  A value that is too high (above 0.5) will result in sub-optimum
        /// performance.
        /// </param>
        public CodeGenerator(SqlConnection connection, int codeLength, int batchSize, double collisionThreshold)
        {
            if (collisionThreshold >= 1.0)
                throw new ArgumentOutOfRangeException("collisionThreshold", collisionThreshold, "must be less than 1");

            _connection = connection;
            CodeLength = codeLength;
            _batchSize = batchSize;
            _collisionThreshold = collisionThreshold;

            _builder = new StringBuilder(codeLength + 1);

            _command = _connection.CreateCommand();
            _command.CommandText = Query;

            _batchParameter = _command.Parameters.Add("@batch", SqlDbType.Structured);
            _batchParameter.TypeName = "dbo.VoucherCodeList";
        }

        public void Dispose()
        {
            if (_disposed)
                return;

            _command.Dispose();
            _disposed = true;
        }

        /// <summary>
        /// Generates unique random codes and inserts them into the database.
        /// </summary>
        /// <param name="numberOfCodes">The number of codes you need.</param>
        /// <returns>A list of unique random codes.</returns>
        public ICollection<string> GenerateCodes(int numberOfCodes)
        {
            var result = new List<string>(numberOfCodes);

            while (result.Count < numberOfCodes)
            {
                var batchSize = Math.Min(_batchSize, numberOfCodes - result.Count);
                var batch = GetBatch(batchSize);
                var oldResultCount = result.Count;
                
                result.AddRange(FilterAndSecureBatch(batch));

                var filteredBatchSize = result.Count - oldResultCount;
                var collisionRatio = ((double)batchSize - filteredBatchSize) / batchSize;

                if (collisionRatio > _collisionThreshold)
                    CodeLength++;
            }

            return result;
        }

        private IEnumerable<string> GetBatch(int batchSize)
        {
            for (var i = 0; i < batchSize; i++)
                yield return GenerateRandomCode();
        }

        private string GenerateRandomCode()
        {
            _builder.Clear();
            for (var i = 0; i < CodeLength; i++)
                _builder.Append(AvailableChars[_random.Next(AvailableChars.Length)]);
            return _builder.ToString();
        }

        private IEnumerable<string> FilterAndSecureBatch(IEnumerable<string> batch)
        {
            _batchParameter.Value = batch.Select(x =>
            {
                var record = new SqlDataRecord(BatchMetaData);
                record.SetString(0, x);
                return record;
            });

            using (var reader = _command.ExecuteReader())
                while (reader.Read())
                    yield return reader.GetString(0);
        }

        /// <summary>
        /// Creates the database schema required by the CodeCenerator.
        /// </summary>
        /// <param name="connection">An open connection to the database.</param>
        public static void CreateSchema(SqlConnection connection)
        {
            using (var command = connection.CreateCommand())
            {
                command.CommandText = @"
                    CREATE TABLE dbo.Voucher (
                        Code nvarchar(" + MaxCodeLength + @") COLLATE SQL_Latin1_General_CP1_CS_AS NOT NULL PRIMARY KEY,
                        BatchId uniqueidentifier NOT NULL
                    );
                    CREATE NONCLUSTERED INDEX IX_Voucher ON dbo.Voucher (BatchId ASC);

                    CREATE TYPE dbo.VoucherCodeList AS TABLE (
                        Code nvarchar(" + MaxCodeLength + @") COLLATE SQL_Latin1_General_CP1_CS_AS NOT NULL
                    );";

                command.ExecuteNonQuery();
            }
        }
    }
}

<h1 align="center">Rahul Saxena</h1>

<p align="center">
  <a href="https://www.linkedin.com/in/rahul-saxena-b25b811b2/" target ="_blank"><img src="https://img.shields.io/badge/LinkedIn-blue?style=for-the-badge&logo=linkedin"></a>
  <a href="https://x.com/Saxen68846Rahul?t=FFWk-Soubr0Y7FbyJOButQ&s=09" target ="_blank"><img src="https://img.shields.io/badge/Twitter-black?style=for-the-badge&logo=x"></a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Views-1.2K-yellow?style=for-the-badge">
  <img src="https://img.shields.io/badge/Stars-2K-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/Follow-95-blue?style=for-the-badge">
  <img src="https://img.shields.io/badge/Visitors-1K-purple?style=for-the-badge">
</p>

---

## 👨‍💻 **About Me**  
🔹 **Backend Developer** | 🔹 **Java & Web Enthusiast** | 🔹 **Open Source Contributor**  

- 🔭 I’m currently working on **Java-based projects**  
- 🌱 I’m learning **React.js, Node.js, **  
- 💬 Ask me about **Java, APIs, and System Design**  
- 📫 Connect with me on [LinkedIn](https://www.linkedin.com/in/rahul-saxena-b25b811b2/)  

---

## 🚀 **GitHub Stats**  
<p align="center">
  <img src="https://github-readme-stats.vercel.app/api?username=Rahul1998sys&show_icons=true&theme=radical" alt="GitHub Stats">
  <img src="https://github-readme-streak-stats.herokuapp.com/?user=Rahul1998sys&theme=radical" alt="GitHub Streak">
  <img src="https://github-readme-stats.vercel.app/api/top-langs/?username=Rahul1998sys&layout=compact&theme=radical" alt="Top Languages">
</p>

