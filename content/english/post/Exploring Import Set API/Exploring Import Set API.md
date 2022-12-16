**This is a revival of a post from my previous blog, there may be some new functionality or features that I didn't cover, but I did go through and test this again to make sure it works on Tokyo, enjoy!**

I want to discuss an API that I haven't really used much in my career as a ServiceNow developer, but I feel deserves some attention and love as it's a really powerful tool: The Import Set API. Prior to this use case, I have not really explored the Import Set API too much. I've seen it used out in the wild for pretty complex use cases when working with clients. I needed a solution that would scale to multiple departments, allowing for specific use cases as needed, and would also be a low-code solution to maintain by other developers in the future. I did not go with Scripted REST API, as I did not want to script/code for various department use cases and maintain multiple scripts. Either solution is great, and much better than granting write access to tables via the Table API.

  
This is a super simple use case of inserting a record into the Incident table and using one Transform map. Before I jump into set up, I want to list out the reasons why I chose to go this route.

  
1. I did not want to provide users/service accounts with the Rest role or any other write roles, the least amount of roles I can provide these users the better. For my use case I only had to provide the import_transformer role.
2. I did not want users writing directly to Incident, as they could put in whatever values they wanted, bypassing any Client Side logic I may have put in place. The fact that the users are writing to an Import Set Table for staging puts my mind at ease, and allows me to control how data is getting into Incident. For example, granting Table API write access also allows access to the PUT/PATCH methods, which allows users to potentially overwrite other users records.
3. In the case there is a flood of inbound API calls that were not intended, I wanted the flexibility to either put some kind of threshold Business Rule on my Import Set Table to deactivate the Transform Map or some kind of indication to do this manually. If I were to allow Table API access I would just get a flood into my Incident table and while I can lock out accounts, etc. I still have to go back and clean up the Incident table, as well as handle erroneous notifications that have gone out. In the case of the staging table I can always go in there and clean up data (within 7 days) and run the Transform on the known good data after clean up.
4. For troubleshooting purposes I can easily look at the staging table and see what kind of data is coming in. For developers who are new to APIs and JSON, this will be helpful to them as well.
5. I could have done this with a Scripted REST API, and have some SRAPIs for GET calls from the Incident table, but I built this with the assumption that multiple departments within the organization will use this, and each will have it's own needs so I wanted something that is scalable and manageable, especially if junior developers need to maintain or expand on this.1. 

## Web Service Import Set

  
For my use case, I'm leaving the [web service import set mode](https://docs.servicenow.com/bundle/rome-platform-administration/page/administer/import-sets/reference/r_ImportSetMode.html) as synchronous, because the other end of the API is a user clicking a button on another system, I don't anticipate a large volume of calls on a daily basis and my import sets will be pretty small in size. I have seen larger implementations of this that do have a multitude of calls, and using the [asynchronous](https://support.servicenow.com/kb?id=kb_article_view&sysparm_article=KB0781666) mode would be a better idea in that case. It really all depends on the amount of data your organization has coming in, and when it needs to be loaded into the platform. If the data does not need to be transformed right away (nightly loads, etc.) then asynchronous would be a better option for this as the transform can be scheduled.

To set up my web service import set, I navigated to System Web Services > Inbound > Create New. Since this is a proof of concept I named it Blog Incident, set the Target table to Incident, and checked Create transform map. I did not check Copy fields from target table because it would copy over 100+ fields from the Incident table, and I only want to set up a few, but this is an option. If you're wondering how you access after the fact, you'll notice that under the Inbound module in the Application Navigator a new module named after your table will appear automatically. Here's what you get after creating it, and before setting up the fields.

![[Create Web Service.png]]

I set up the following fields but you can really send anything, making sure for Choice fields that you send the value in the API call and not the Label. For fields such as Caller and Assignment group, either the API will need to send the sys_id value, or a field script would need to be used to find the values. I'll show an example in the transform map. If you've set up Data Sources and transform maps before, think of the Web Service Fields as the 'source' field i.e. the field you're sending from the excel, etc. except they'll be coming in from a JSON request body in an API call. In fact, in transform maps these fields are referred to as source.field_name (source.short_description) just like in Data Sources. Note: Be sure to expand the length of your field for things such as Short description and Description so they're not truncated at the Import Set Table level. I set both of mine to 1000, though the field length on Short description on Incident itself will truncate the characters.

![[Web Service Fields.png]]


## Transform Map

In creating the Web Service a Transform map was created if the box was checked, but some field mapping has to be done. This is a very basic example of using a transform map on the Web Service Import Set, but it can be expanded on just like any other transform map. Robust Import Set Transformers are also an option when using Web Service Import Sets because they also key off the Source table (such as the Import Set Table).

In the transform map I mapped the following directly, and going on the assumption the API will send the sys_id of the assignment group, but a field script could be set up for other values:


``` 
u_short_description = Short description

u_description = Description

u_assignment_group = Assignment group with Choice action set to ignore.

  

I created field scripts for the following:

Contact type with Choice action set to ignore.

answer = (function transformEntry(source) {

    return 'self_service'; // return the value to be put into the target field

})(source);

Caller with Choice action set to ignore.

answer = (function transformEntry(source) {

    //In the example the API is sending the email of the user, but this can be anything such as user_name or other key values stored in the sys_user table.

    var email = source.u_caller;

    var userGr = new GlideRecord('sys_user');
    userGr.addQuery('email', email);
    userGr.query();
    if (userGr.next()) {
        var sysId = userGr.sys_id;
    }
    return sysId; // return the value to be put into the target field

})(source);
```

![[Transform Map.png]]
  


## REST API Explorer

Finally I'll test this out with the REST API Explorer. I do want to note that I am doing this as admin in my PDI, but if you're using a service account or other user to send the API call, that user will need the import_transformer role. This is the only role they will need to use the Import SET API to insert records, they don't need the rest roles, or itil, or anything else, which is another reason I love using this API rather than something like Table API. There is a related link at the bottom of the Web Service record to jump into the Rest API Explorer that will load the table automatically. It would be nice if REST API Explorer would pop in the Web Service Fields automatically as it can detect the table, but you have to do this manually in the builder tab.

![[REST API Table.png]]

![[Request Headers.png]]

![[Request Body.png]]

Here's the JSON body i'm using:

```{"u_short_description":"This is my short description!","u_caller":"admin@example.com","u_description":"This is my long description, it's way longer than my short description.","u_assignment_group":"d625dccec0a8016700a222a0f7900d06"}```

I received a 201 response and this response body:

```{
    "import_set": "ISET0010002",
    "staging_table": "u_blog_incident",
    "result": [
    {
        "transform_map": "Blog Incident",
        "table": "incident",
        "display_name": "number",
        "display_value": "INC0010094",
        "record_link": "https://myinstance.service-now.com/api/now/table/incident/3071162d2f284510e0374f2e2c99b6c1",
        "status": "inserted",
        "sys_id": "3071162d2f284510e0374f2e2c99b6c1"
    }
]
}
```

Finally here's the Incident:

![[Incident.png]]


## Takeaways

I really enjoyed using the Import Set API for my use case, as it seems simple, scalable, and secure. Developers with basic knowledge of Transform Maps can set up and maintain them, and they're pretty easy for new developers to learn as well. In my example I kept it very simple with one record being sent, but the option to insert multiple records with the Import Set API is there. I mentioned before that Robust Transform Maps can be used with Web Service Import Sets, and I didn't explore the concept of Concurrent Imports, but the ability to expand on transforms and bring in large sets of data via the Import SET API is there. As I type this I wonder how useful Data Sources and CSV files and pulls of data still are, when on the sender's side of the house an API call with the request body JSON can be sent periodically to ServiceNow and utilize this method. I welcome any commentary on this.

For fun I went ahead and tried the Insert Multiple method out because I was curious. First I had to install the Insert Multiple Web Service plugin on my instance. I then added an entry to the REST Insert Multiple sys_rest_insert_multiple table. Per the Import Set API documentation, the transformation is set to Asynchronous by default. I left it at that because as I mentioned earlier for larger sets of data this would be the preferred mode. I then created a Column mapping on my REST Insert Multiple record with the Type of JSON and the Column mapping of Column name since I want it mapped to the column name of my Web Service Import Set. There's more information about this on the Import Set API documentation.

![[REST Insert Multiple.png]]

In REST API Explorer I clicked on the Insert Multiple Records from same request (POST) option and it provided the following endpoint: https://instance-name.service-now.com/api/now/import/{stagingTableName}/insertMultiple

![[REST Insert Multiple 1.png]]

I then formatted my request body and wrapped my objects inside of an array within an object, I found this example from the Import Set API documentation:

```{ "records":[{"u_short_description":"This is my short description 1!","u_caller":"admin@example.com","u_description":"This is my long description, it's way longer than my short description.","u_assignment_group":"d625dccec0a8016700a222a0f7900d06"}, {"u_short_description":"This is my short description 2!","u_caller":"admin@example.com","u_description":"This is my long description, it's way longer than my short description.","u_assignment_group":"d625dccec0a8016700a222a0f7900d06"}]}```


After I sent the the call I now get a Response Body with the sys_ids of both the Import Set and Multi Import Set (sys_multi_import_set):


```{
    "import_set_id": "d237de2d2f284510e0374f2e2c99b60a",
    "multi_import_set_id": "1a37de2d2f284510e0374f2e2c99b60a"
}
```

And I can see my two incident records were inserted:

![[Incidents.png]]

## Resources

[ServiceNow Product Documentation: Web service import sets](https://docs.servicenow.com/bundle/rome-platform-administration/page/administer/import-sets/concept/c_WebServiceImportSets.html)
[ServiceNow Product Documentation: Import Set API](https://developer.servicenow.com/dev.do#!/reference/api/rome/rest/c_ImportSetAPI)