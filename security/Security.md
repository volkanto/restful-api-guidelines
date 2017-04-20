# Security

## {{ book.must }} Secure Endpoints with OAuth 2.0

Every API endpoint needs to be secured using OAuth 2.0. Please refer to the 
[official OpenAPI spec](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#security-definitions-object)
on how to specify security definitions in you API specification or take a look at the following example.

```yaml
securityDefinitions:
  oauth2:
    type: oauth2
    flow: password
    tokenUrl: https://token.services.auth.zalando.com/oauth2/access_token
    scopes:
      fulfillment-order-service.read: Access right needed to read from the fulfillment order service.
      fulfillment-order-service.write: Access right needed to write to the fulfillment order service.      
```

The example defines OAuth2 with password flow as security standard used for authentication when accessing endpoints; additionally, there are two API access rights defined via the scopes section for later endpoint authorization usage - please see next section.

It makes little sense specifying the flow to retrieve OAuth tokens in the `securityDefinitions` section, as API endpoints should not care, how OAuth tokens were created. Unfortunately the `flow` field is mandatory and cannot be ommited. API endpoints should always set `flow: password` and ignore this information.

## {{ book.must }} Define and Assign Access Rights (Scopes)

Every API needs to define access rights, called scopes here, and every endpoint needs to have at least one scope assigned. Scopes are defined by name and description per API specification, as shown in the previous section. Please refer to the following rules when creating scope names:

```
<api-scope> ::= <api-standard-scope> |            -- should be sufficient for majority of use cases 
                <api-resource-specific-scope> |   -- for special security access differentiation use cases 
                <api-pseudo-scope>                -- used to explicitly indicate that access is not restricted
                
<api-standard-scope>          ::= <application-id>.<access-type> 
<api-resource-specific-scope> ::= <application-id>.<resource-id>.<access-type>
<api-pseudo-scope>            ::= uid

<application-id> ::= <as defined via STUPS>
<access-type>    ::= read | write           -- might be extended in future
<resource-id>    ::= <free identifier following application-id syntax>
```

APIs should stick to standard scopes by default -- for the majority of use cases, restricting access to specific APIs (with read vs. write differentiation) is sufficient for controlling access for client types like merchant or retailer business partners, customers or operational staff. We want to avoid too many, fine grained scopes increasing governance complexity without real value add. In some situations, where the API serves different types of resources for different owners, resource specific scopes may make sense.

Some examples for standard and resource-specific scopes:

| Application ID      | Resource ID      | Access Type | Example                           |
|---------------------|------------------|-------------|-----------------------------------|
| `fulfillment-order` |                  | `read`      | `fulfillment-order.read`          |
| `fulfillment-order` |                  | `write`     | `fulfillment-order.write`         |
| `sales-order`       | `sales_order`    | `read`      | `sales-order.sales_order.read`    |
| `sales-order`       | `shipment_order` | `read`      | `sales-order.shipment_order.read` |

After scopes names are defined and the scope is declared in the security definition at the top of an API specification, it should be assigned to each API operation by specifying a [`security` requirement](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#securityRequirementObject) like this:

```yaml
paths:
  /sales-orders/{order-number}:
    get:
      summary: Retrieves a sales order
      security:
        - oauth2:
          - sales-order-service.sales_order.read
```

In very rare cases a whole API or some selected endpoints may not require specific access control. However, to make this explicit you should assign the `uid` pseudo access right scope in this case. It is the user id and always available as OAuth2 default scope. 

```yaml
paths:
  /public-information:
    get:
      summary: Provides public information about ... 
               Accessible by any user; no access rights needed. 
      security:
        - oauth2:
          - uid
```

Hint: you need not explicitly define the "Authorization" header; it is a standard header so to say implicitly defined via the security section.


## {{ book.must }} Document supported access scenarios
A service in the Zalando platform may support a couple of different access scenarios to support different use cases. Client and API endpoint developers must understand the different access scenarios and act accordingly: Client developers must send the required tokens depending on the access scenario they want to use. API endpoint developers must implement one or more access scenarios and document the supported access scenarios in order to help client developers to implement their clients.

The following access scenarios exist:

<b>`service`</b>:
Service to service communication to the Zalando platform or between two non-platform services. In this access scenario a service token (claim `"https://identity.zalando.com/realm": "services"`) must be send to the API endpoint carrying the required privileges to access the requested resource. The token must be send as a bearer token in the `Authorization` HTTP header. Data segregation based on business partners must be enforced: The business partner carried in the token (claim `https://identity.zalando.com/bp`) must be equal to the business partner owning the requested resource.

<b>`inner-platform`</b>:
A Zalando platform service accessing a REST API of another service within the platform. This access scenario is basically the same as the `service` access scenario, except that platform services may get access to data of all business partners: The service token must therefore not only carry the required API-permissions, but must also carry the privilege `z::_platform` in order to access data of all business partners within the platform.

<b>`customer`</b>:
This scenario describes a use case, where a service having direct customer interaction, calls a service of the platform to access customer data of this very customer (e.g. a customer interacting with a web shop). In this case two tokens must be send to the platform API endpoint: A service token of the service doing the call and a customer token identifying the customer (via a custom `X-Customer-Token` HTTP header field). The service token must carry the required privileges to access the API, the customer token is used to identify the customer. The platform service must only grant access to resources belonging to the business partner defined in the service token and to the customer carried in the the customer token (claim `iat`). If the business partner is not the owner of the data, the the customer token must carry an appropriate consent to access the requested customer data (claim `http://identity.zalando.com/approval`).

<b>`customer-all`</b>:
The customer-all access scenario is designed for use cases where a service calling a platform service must have access to the data of all customers of one business partner. These are non-customer interactive use cases in general. Consumer facing applications usually have customer-interactive use cases and should use the customer access scenario instead (only having access to the data of the customer the service is interacting with). In the customer-all access scenario, the client service must send a service token to the platform carrying the required privileges to access this API and carry a privilege named 'z::_all-customer-data' to gain access to the data of all customers of the business partner.

<b>`impersonation`</b>:
The impersonation access scenario supports use cases where customer care agents interact with e.g. a web shop in the name of the customer. Therefore the customer care agent must request a special user token carrying the impersonation information (which customer the agent is impersonating). This information is carried in the claim `http://identity.zalando.com/impersonating` in the user token. Client services must then send two tokens: The user token (send along in the `X-Customer-Token` HTTP header and carrying the impersonation information) and the service token. API endpoints supporting this access scenario must give access to the resources owned by the business partner carried in the service token and the customer referred to in the impersonation token.

<b>`user`</b>:
This access scenario is used in use cases where developers access a REST API directly (e.g. for debugging during an incident scenario). In this use case the user must send its user token as a bearer token in the 'Authorisation' HTTP header carrying the required permissions to access a resource of an API. The API must only hand out data belonging to the business partner carried in the user token, unless the token carries the privilege `z::_platform` which grants access to the data of all business partners.

API provides must specify the supported access scenarios for each operation in their OpenAPI definitions using the custom 'x-access-scenarios' parameter. Valid values are `service`, `inner-platform`, `customer`, `customer-all`, `impersonation` and `user`. An example for such a definition would be:

```
paths:
  /my-resource/{id}:
    parameters:
      - $ref: '#/parameters/id'

	get:
      responses:
        200:
          schema:
            $ref: '#/definitions/MerchantMyResource'
      security:
        - oauth2: ['z::my-permission.read']
      x-access-scenarios: ['inner-platform','service','customer','customer-all','impersonation','user']
```