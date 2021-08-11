# BigConnect Explorer AML Demo

This example includes several source files and queries that illustrate an Anti-Money Laundering (AML) use case, 
where we try to identify some Fraud Rings using graph technology.

Included are data files (csv format) for People, Companies, Addresses, Bank Accounts, Shares of company stock, and financial Transactions.

## Loading the data
Open Cypher Lab and type in the following Cypher queries to load the data:

1. Load transactions and their Bank Accounts (takes a while):

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/transactions.csv' AS row
        WITH row
        CREATE (b1:bankAccount { number: row.account1 })-[:originated]->(tx:transaction)-[:beneficiary]->(b2:BankAccount { number: row.account2 })
        SET tx.amount = toFloat(row.amount), tx.date = date(row.tx_date), tx.txid = row.tx_id
        
2. Load transactions and link them to Bank Accounts (takes a while)::

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/transactions.csv' AS row
        WITH row
        MATCH (b1:bankAccount { number: row.account1 })
        MATCH (b2:bankAccount { number: row.account2 })
        CREATE (b1)-[:originated]->(tx:transaction)-[:beneficiary]->(b2)
        SET tx.amount = toFloat(row.amount), tx.date = date(row.tx_date), tx.txid = row.tx_id

3. Load Addresses:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/addresses.csv' AS row
        WITH row
        CREATE (a:address) 
        SET a.eid = row.id, a.city = row.city, a.country = row.country, a.geoLocation = toGeoPoint(row.lat+','+row.long), a.zipCode = row.zip_code, a.street = row.street

4. Load Companies:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/companies.csv' AS row
        WITH row
        CREATE (a:organization) 
        SET a.eid = row.id, a.name = row.name, a.description = row.description

5. Load Persons:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/people.csv' AS row
        WITH row
        CREATE (a:person) 
        SET a.eid = row.id, a.firstName = row.first_name, a.lastName = row.last_name, a.email = row.email, a.country = row.country

6. Load and link Companies with their Addresses:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/companies_addresses.csv' AS row
        WITH row
        MATCH (c:organization { eid: row.company })
        MATCH (a:address { eid: row.address })
        CREATE (c)-[:hasAddress]->(a)
        
7. Load and link People with their Addresses:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/people_addresses.csv' AS row
        WITH row
        MATCH (c:person { eid: row.person })
        MATCH (a:address { eid: row.address })
        CREATE (c)-[:hasAddress]->(a)
        
8. Load and link Companies with their Bank Accounts:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/companies_accounts.csv' AS row
        WITH row
        MATCH (c:organization { eid: row.company })
        MATCH (b:bankAccount { number: row.account })
        CREATE (c)-[:hasBankAccount]->(b)

9. Load and link Persons with their Bank Accounts:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/people_accounts.csv' AS row
        WITH row
        MATCH (c:person { eid: row.person })
        MATCH (b:bankAccount { number: row.account })
        CREATE (c)-[:hasBankAccount]->(b)

10. Load and link Companies with other Companies:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/companies_shares.csv' AS row
        WITH row
        MATCH (c1:organization { eid: row.company1 })
        MATCH (c2:organization { eid: row.company2 })
        CREATE (c1)-[h:holds]->(c2) SET h.share = toFloat(row.share)

11. Load and link Persons with Companies:

        USING PERIODIC COMMIT 1000
        LOAD CSV WITH HEADERS FROM 'https://github.com/bigconnect/demos/raw/master/explorer/aml/people_shares.csv' AS row
        WITH row
        MATCH (p:person { eid: row.person })
        MATCH (c:organization { eid: row.company })
        CREATE (p)-[h:holds]->(c) SET h.share = toFloat(row.share)

12. Create affiliations based on common addresses:

        MATCH(e1)-[:hasAddress]->(a:address)<-[:hasAddress]-(e2) 
        WHERE id(e1) < id(e2)
        CREATE (e1)-[:hasAffiliation]->(e2)

13. Create affiliations based on shares owned:

        MATCH(e)-[h:holds]->(c:organization) 
        WHERE h.share >= 50 
        CREATE (e)-[:hasAffiliation]->(c)

14. Find the first possible 10 fraud rings:

        MATCH(p1:person)-[:hasAffiliation*]-(t1)-[:hasBankAccount]->(a1)-[:originated]->(tx)-[:beneficiary]-(a2)-[:hasBankAccount]-(t2)-[:hasAffiliation*]-(p2:person) 
        WHERE id(p1) < id(p2) 
        WITH p1, p2, COUNT(*) as paths, SUM(tx.amount) as total
        RETURN p1.firstName+' '+p1.lastName as Person1, p2.firstName+' '+p2.lastName as Person2, total*paths*paths as FraudScore
        ORDER BY FraudScore DESC
        LIMIT 10
        
