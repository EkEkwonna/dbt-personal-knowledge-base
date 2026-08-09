# Referencing Sources

In dbt we can utilise the `.yml` file to reference a datasource with the use of language such as 

~~~
{{source('schema_name','object_name')}}
~~~

This will create a direct reference to the data souce.

dbt will use the yml file to identify the reference of the data source. 

This means if we have several models using a specific data source. We can make the change in the yml file and dbt will compile all the models accordingly (instead of making changes manaually)



