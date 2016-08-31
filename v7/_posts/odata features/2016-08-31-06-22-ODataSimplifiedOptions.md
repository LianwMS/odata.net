---
layout: post
title: "Centralized control for OData Simplified Options"
description: ""
category: "5. OData Features"
---

In previous OData library, the control of writing/reading/parsing key-as-segment uri path is separated in ODataMessageWriterSettings/ODataMessageReaderSettings/ODataUriParser. The same as writing/reading OData annotation without prefix. From ODataV7.0, we add a centialized control class for them, which is ODataSimplifiedOptions.



According to the spec, a navigation property can be used in multiple bindings with different path. It makes a lot of sense for navigation property under complex and containment. From ODataLib V7.0, we are able to support it.
The valid format of a binding path is:
{% highlight csharp %}
[ ( qualifiedEntityTypeName / qualifiedComplexTypeName ) "/" ] 
                    *( ( complexProperty / complexColProperty) "/" [ qualifiedComplexTypeName "/" ] ) 
                    *( ( containmentNav / colContainmentNav ) "/" [ qualifiedEntityTypeName "/" ] )
                    navigationProperty
{% endhighlight %}

For example, we have containment in [Trippin service](http://services.odata.org/V4/(S(qqntzoewadope25a3bh2d5bi))/TripPinServiceRW/$metadata)：

Person <br />
&nbsp;&nbsp;|---- Trips (containment) <br />
&nbsp;&nbsp;&nbsp;&nbsp;|---- PlanItems (containment) <br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|---- NS.Flight (derived type of PlanItem) <br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|---- Airline (navigation property) <br />

Now we bind entityset `Airlines` to the property `Airline`. Since for navigation property binding, we need start from non-containment entityset, then route to navigation property with binding path. So we should have binding:
{% highlight csharp %}
<EntitySet Name="People" EntityType="NS.Person">
    <NavigationPropertyBinding Path="Trips/PlanItems/NS.Flight/Airline" Target="Airlines"/>
</EntitySet>
{% endhighlight %}
If we have `InternationalTrips` under `Person` which has type `Trip` as well, we can have binding `<NavigationPropertyBinding Path="InternationalTrips/PlanItems/NS.Flight/Airline" Target="Airlines"/>` under `People` as well.
Please note that current Trippin implementation for this part does not have strict compliance with spec due to prohibition of breaking changes.

Let's have another example of complex which have multi bindings. `City` is a navigation property of complex type `Address`, and `Person` has `HomeAddress`, `CompanyAddress` which are both `Address` type. Then we can have binding:
{% highlight csharp %}
<EntitySet Name="People" EntityType="Sample.Person">
    <NavigationPropertyBinding Path="HomeAddress/City" Target="Cities1" />
    <NavigationPropertyBinding Path="CompanyAddress/City" Target="Cities2" />
</EntitySet>
{% endhighlight %}

For single binding, binding path is just the navigation property name or type cast appending navigation property name. But for multiple bindings, binding path becames an essential info to create a binding.
As a result, following APIs are added:

### EDM ###
`public EdmNavigationPropertyBinding (IEdmNavigationProperty navigationProperty, IEdmNavigationSource target, IEdmPathExpression bindingPath)` <br />
Use this API to create EdmNavigationPropertyBinding instance if the bindingpath is not navigation property name.

 `public void AddNavigationTarget (IEdmNavigationProperty navigationProperty, IEdmNavigationSource target, IEdmPathExpression bindingPath)` <br />
Add a navigation property binding and specify the whole binding path.

`public virtual Microsoft.OData.Edm.IEdmNavigationSource FindNavigationTarget (IEdmNavigationProperty navigationProperty, IEdmPathExpression bindingPath)` <br />
Find navigation property with its binding path.
	
### ODL ###
`public ODataQueryOptionParser (IEdmModel model, ODataPath odataPath, String queryOptions)` <br />
`public ODataQueryOptionParser(IEdmModel model, ODataPath odataPath, IDictionary<string, string> queryOptions, IServiceProvider container)` <br />
Possibly need ODataPath to resolve navigation target of segments in query option if the navigation property binding path is included in both path and query option. Refer: [Navigation property under complex type](http://luoyan0517.github.io/odata.net/v7/#06-18-navigation-under-complex).

Take the above complex scenario for example. For generating this kind of model, we need use the new AddNavigationTarget API, and different navigation target can be specified:
{% highlight csharp %}
people.AddNavigationTarget(cityOfAddress, cities1, new EdmPathExpression("HomeAddress/City"));
people.AddNavigationTarget(cityOfAddress, cities2, new EdmPathExpression("Addresses/City"));
{% endhighlight %}
`cityOfAddress` is the variable to present navigation property `City` under `Address`, and `cities1` and `cities2` are different entityset based on entity type `City`.

To achieve the navigation target, the new FindNavigationTarget API can be used:
{% highlight csharp %}
IEdmNavigationSource navigationTarget = people.FindNavigationTarget(cityOfAddress, new EdmPathExpression("HomeAddress/City"));
{% endhighlight %}
