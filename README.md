# Testing plv8 stored procedure to fix jsonb data

```
docker pull clkao/postgres-plv8
docker run -d --name mypostgres clkao/postgres-plv8
docker run -it --rm --link mypostgres:postgres clkao/postgres-plv8 psql -h postgres -U postgres
```

```sql
create database stevek;
\c stevek
create extension plv8;
create table foo ( data jsonb );
CREATE UNIQUE INDEX foo_id_idx ON foo (((data->>'id'::text)::bigint));
ALTER TABLE foo ADD CONSTRAINT foo_id_nn CHECK ((data->>'id') IS NOT NULL); 

do language plv8 $$
    'use strict';

    function getRandomInt(min, max) {
        min = Math.ceil(min);
        max = Math.floor(max);
        return Math.floor(Math.random() * (max - min)) + min;
    }

    for (let id = 0; id < 99; id++) {
        const data = {
            id: id,
            foo: {
                bar: getRandomInt(1, 100)
            }
        };
        plv8.execute('INSERT INTO foo VALUES($1)', data);
    }
$$;

do language plv8 $$
    'use strict';

    // Here is the point of using plv8: fixing jsonb data can be done in javascript
    function fix(data) {
        data.fixed = data.fixed || 0;
        data.fixed++;
        data.qux = {quxx: data.foo.bar};
    }

    const plan = plv8.prepare('SELECT data FROM foo');
    const cursor = plan.cursor();
    const batchSize = 10;
    let rows;
    while (rows = cursor.fetch(batchSize)) {
        rows.forEach(function (row) {
            const data = row.data;
            plv8.elog(NOTICE, `BEFORE: ${JSON.stringify(data)}`);
            fix(data);
            plv8.elog(NOTICE, `AFTER: ${JSON.stringify(data)}`);
            plv8.execute(`UPDATE foo SET data = $1 WHERE (data->>'id')::bigint = $2`, data, data.id);
        });
    }
    cursor.close();
    plan.free();
$$;
```

## Notes
* This will be one massive transaction but googling around (e.g. [this](http://stackoverflow.com/questions/709708/maximum-transaction-size-in-postgresql)) suggests maybe that's okay, unlike in Oracle where it'd build up a massive amount of redo.
* The following helped understand transactions and subtransactions a bit: http://stackoverflow.com/a/25428060/296829


