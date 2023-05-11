Questions on postgres_fdw
=========================

1. For sum with numeric, gpdb7 need array(count(c1), sum(c1).

Only expect sum(c1) without count(c1)


2. For avg with float, double, gpdb7 needs array(count(c1), sum(c1), count(c1) * var_pop(c1)]

Only expect sum(c1), count(c1) without count(c1) * var_pop(c1)

---
### **Answer for Question 1,2:**

For sum(c1::numeric), avg(c1::float) and avg(c1::double), what array we make depends on their
corresponding aggregate transition state.

sum(c1::numeric) --> [sum(c1), count(c1)]
   - Please look at numeric_avg_combine(), which is sum(numeric)'s aggcombinefn.

avg(c1::float) and avg(c1::double) --> [N, Sx, Sxx] ([count(c1), sum(c1), count(c1) * var_pop(c1)])
   - Please look at float8_combine(), which is avg(c1::float) and avg(c1::double)'s aggcombinefn.

Make Remote SQL according to aggregate transition state will simplify processing data returned from remote server.
___

3. For example SQL, 

```
SELECT avg(c1), avg(c2) from table where c1 > 3;
```

the generated SQL is as below,

```
SELECT array[count(c1) filter (where c1 > 3), sum(c1) filter (where  c1 > 3)], 
array[count(c2) filter (where c1 > 3), sum(c2) filter(where c1 > 3)] from table where c1 > 3;
```

The expect SQL is as below,

```
SELECT array[count(c1), sum(c1)], array[count(c2), sum(c2)] from table where c1 > 3
```

---
### **Answer for Question 3:**

You can install our branch and have a try. (https://github.com/jingwen-yang-yjw/gpdb/blob/spike_mpp_pushdown)

We won't append filters in WHERE clause to aggregates. Please look at these examples as follows.
```
testdb=# explain verbose SELECT avg(a), avg(b) from multi_table where a > 3;
                                                       QUERY PLAN
------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=1419.86..1419.87 rows=1 width=64)
   Output: avg(a), avg(b)
   ->  Gather Motion 4:1  (slice1; segments: 4)  (cost=243.50..1419.84 rows=4 width=64)
         Output: (PARTIAL avg(a)), (PARTIAL avg(b))
         ->  Foreign Scan  (cost=243.50..1419.79 rows=1 width=64)
               Output: (PARTIAL avg(a)), (PARTIAL avg(b))
               Relations: Aggregate on (public.multi_table)
               Remote SQL: SELECT array[count(a), sum(a)], array[count(b), sum(b)] FROM public.test_fdw WHERE ((a > 3))
 Optimizer: Postgres query optimizer
(9 rows)

testdb=# explain verbose select sum(a) filter(where b < 5) from multi_table where a < 2;
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=1419.84..1419.85 rows=1 width=8)
   Output: sum(a) FILTER (WHERE (b < 5))
   ->  Gather Motion 4:1  (slice1; segments: 4)  (cost=243.50..1419.83 rows=4 width=8)
         Output: (PARTIAL sum(a) FILTER (WHERE (b < 5)))
         ->  Foreign Scan  (cost=243.50..1419.78 rows=1 width=8)
               Output: (PARTIAL sum(a) FILTER (WHERE (b < 5)))
               Relations: Aggregate on (public.multi_table)
               Remote SQL: SELECT sum(a) FILTER (WHERE (b < 5)) FROM public.test_fdw WHERE ((a < 2))
 Optimizer: Postgres query optimizer
(9 rows)
```
___


4. In transcoding.c, a lot of code copied from numeric.c.  Can you expose the data structure and functions in numeric.h or new header file numeric_api.h instead?

Suggested APIs

```

Datum int64_avg_aggstate_create(PG_FUNCTION_ARGS)
{
	int64 count = PG_GETARG_INT64(0);
	int64 sum = PG_GETARG_INT64(1);


	return serialized_polynumaggstate;
}


Datum int128_avg_aggstate_create(PG_FUNCTION_ARGS)
{
	int64 count = PG_GETARG_INT64(0);
	int128 sum = *((int128 *) PG_GETARG_POINTER(1));

	return serialized_polynumaggstate;
}

Datum numeric_avg_aggstate_create(PG_FUNCTION_ARGS)
{
	int64 count = (int64) PG_GETARG_INT64(0);
	Numeric sum = PG_GETARG_NUMERIC(1);

	return serialized_numericaggstate;
}

Datum float8_avg_aggstate_create(PG_FUNCTION_ARGS)
{
	int64 count = PG_GETARG_INT64(0);
	float8 sum = PG_GETARG_FLOAT8(1);

	return arraytype;
}

Datum int64_sum_aggstate_create(PG_FUNCTION_ARGS)
{
	int64 sum = PG_GETARG_INT64(0);

	return serialized_polynumaggstate;
}


Datum int128_sum_aggstate_create(PG_FUNCTION_ARGS)
{
	int128 sum = *((int128 *) PG_GETARG_POINTER(0));

	return serialized_polynumaggstate;
}

Datum numeric_sum_aggstate_create(PG_FUNCTION_ARGS)
{
	Numeric sum = PG_GETARG_NUMERIC(0));

	return serialized_numericaggstate;
}

```


---
### **Answer for Question 4:**

Thanks for your suggestion. We'll try to improve it later.

But such code changes about gpdb might need more reviews and approvals.
___


5. Average for float and double is not done yet.

You can use this data structure to treat as ArrayType that contain three double values


```
typedef struct ExxFloatAvgTransdata
{
        ArrayType arraytype;
        int32   nelem;
        float8  data[3];     // float8[3]
} ExxFloatAvgTransdata;

ExxFloatAvgTransdata avgtrans;
ExxFloatAvgTransdata *tr0;

tr0 = &avgtrans;
SET_VARSIZE(tr0, sizeof(ExxFloatAvgTransdata));
tr0->arraytype.ndim = 1;
tr0->arraytype.dataoffset = (char*) tr0->data - (char*)tr0;
tr0->arraytype.elemtype = FLOAT8OID;
tr0->nelem = 3;
tr0->data[0] =  N;
tr0->data[1] =  sumX;
tr0->data[2] = sumX * sumX;


ArrayType *arr = (ArrayType*) tr0;

```

Do ArrayType need serialize function? ArrayType is already a varlena type and wondering it is already serialized?


---
### **Answer for Question 5:**

We have done avg(c1::float) and avg(c2::double).

Aggregates avg(c1::float) and avg(c2::double) don't need to serialize. Because we build Remote SQL according to their aggregate transition state. (Please refer to Answer for Question 1,2.)

We have added related test cases about them in our branch(https://github.com/jingwen-yang-yjw/gpdb/blob/spike_mpp_pushdown/contrib/postgres_fdw/sql/mpp_gp2pg_postgres_fdw.sql).
___


6. How to get the Segment ID and total number of segments?


---
### **Answer for Question 6:**

You can refer to this commit (https://github.com/greenplum-db/gpdb/commit/413719afb320ff21a8723dc381fb111653a35368).

In this commit, we can get the Segment ID and total number of segments in function postgresBeginForeignScan().
___