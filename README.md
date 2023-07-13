<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://www.curbit.com/">
    <img src="https://uploads-ssl.webflow.com/60066c7287d96dc62123c966/63334f79313aa01d173332ce_curbit%20logo%404x-p-800.png" alt="Logo" width="80" height="60">
  </a>
<h3 align="center">Notification center</h3>
  <p align="center">
   <a href="https://excellent-tiara-b60.notion.site/Full-stack-engineer-assessment-0cb6cb5171bf4f6b8e0ea71ee0a5a436"><strong>Assessment Curbit</strong></a>
  </p>
</div>


<!-- ABOUT THE PROJECT -->
## About The Project

Curbit needs to develop an ordering system that processes orders from different restaurants

## Context

![image](https://github.com/4dagio/assessment/assets/3275936/a23a4b68-9053-4999-9161-f46c278f660e)


### Process description

1. Restaurant generate orders from differents sources like: POS or KDC
2. Orders add to service bus 
3. Services OrdersEngine read the messages from queue to process approach the rules and status
4. Service NotificationEngine read the messages from queue to update Order and notify to users when status is: 'prepared'||'ready'


### Built With

* [[C#]](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/local-functions)
* [[Azure]](https://azure.microsoft.com/en-us/solutions/open-source/?&ef_id=_k_Cj0KCQjwnrmlBhDHARIsADJ5b_mwBe0ocBrasJ-l6VPy9GryG6Kxal_q1vXCt9A9QKQIGYdcbfNIAzgaAqBGEALw_wcB_k_&OCID=AIDcmm3804ythc_SEM__k_Cj0KCQjwnrmlBhDHARIsADJ5b_mwBe0ocBrasJ-l6VPy9GryG6Kxal_q1vXCt9A9QKQIGYdcbfNIAzgaAqBGEALw_wcB_k_&gclid=Cj0KCQjwnrmlBhDHARIsADJ5b_mwBe0ocBrasJ-l6VPy9GryG6Kxal_q1vXCt9A9QKQIGYdcbfNIAzgaAqBGEALw_wcB)

## High-level architecture

![image](https://github.com/4dagio/assessment/assets/3275936/60457db3-8be1-4938-907e-833ee7b62fd8)

### Notes: it is assumed that:

1. The data source has the capacity to call rest or soap services.
2. The amount of transactions are within the capacity of the premiere layer of the azure api management (approx. 100,000,000 calls each month).
3. Notifications will be sent to mobile devices.
4. Observability will be enabled with AppInsights and Monitor.

### Explanation
In this case, I include terms such as Order, Restaurant, User, Status, Queue, Notification.

- **Api Management:** The POS/KDS system makes a HTTP POST request to an endpoint on the API Management service, passing the order data in the request body. This data could include details such as restaurant_id, source_id, order_id, store_name, user_name, and time_ready. With the api management we can validate data and verifying the source of the request
  
- **Send to service bus:** If the order data is valid and the request is authorized, the API Management service forwards the order data to the Azure Service Bus. This could involve serializing the order data into a message format that the Service Bus can handle (such as JSON), and then sending the message to the appropriate queue.

- **Service Bus Processing:** The Azure Service Bus receives the message and places it in a queue to be processed. This queue could be monitored by a separate service or function, which is responsible for processing the orders.
    
    - **Enable duplicate message detection** To ensure that duplicate orders are not created we can enable duplicate detection for a queue. Azure Service Bus keeps a history of all messages sent to the queue for a configure amount of time. During that interval, your queue or topic won't store any duplicate messages. Enabling this property guarantees exactly once delivery over a user-defined span of time  

- **Response to POS/KDS:** After the message has been successfully sent to the Service Bus, the API Management service sends a HTTP response back to the POS/KDS system to confirm that the order was received and is being processed with the code 202 Accepted.

- **Function OrderEngine:**  It's responsible for accepting orders and persisting orders to the cosmos Orders database.

- **Function OrderNotification:** The patch function is responsible for managing the lifecycle of an order once it has been placed and persisted to the Cosmos database. It would have the following responsibilities:

    - **Message Handling:** This context would listen for new messages on the Update queue in Azure Service Bus. These messages would be generated by other parts of your system (like the Order Processing context), and they would represent some kind of change that needs to be applied to an existing order, such as a status update. Each message would need to contain enough information to identify the order that it relates to: source_id, restaurant_id, order_id and the nature of the update that should be applied (e.g., a new status).

    - This is the function within the context that carries out the business logic related to order updates. It might handle the extraction of data from incoming messages, apply business rules, depends of the new state: placed or ready to go. 

    - Data Updating: The repository layer would communicate with the Cosmos DB, apply the necessary update (like changing the order's status), and handle any errors that might occur (like if no order with the given order_id exists).

- **Event Notification:** This event emit domain events to signal that an order has been updated. These events send a trigger to notification hub for sending a notification to the user with the updates.

  - The "Order Updating" bounded context would also need to include mechanisms for error handling and logging. For example, if an update message is received for an order that doesn't exist in the database, the system should log an error and handle it gracefully. This could involve sending a message back to the queue to retry later, or moving it to a "dead-letter" queue for manual investigation.



<!-- GETTING STARTED -->
## Codebase

For development this project we can use **Hexagonal Architecture**, is driven by the idea that the application is central to your system. All inputs and outputs reach or leave the core of the application through a port that isolates the application from external technologies, tools and delivery mechanics
I can use this template to codeBase:
[Hexagonal Architecture](https://github.com/Amitpnk/Hexagonal-architecture-ASP.NET-Core)


```cs
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Host;
using Microsoft.Extensions.Logging;
using Microsoft.Azure.Cosmos;
using System.Threading.Tasks;
using Newtonsoft.Json;

namespace Company.Function
{
    public static class OrderEngine
    {
        private static readonly string EndpointUri = "uri-test";
        private static readonly string PrimaryKey = "primary-key";
        private static CosmosClient cosmosClient = new CosmosClient(EndpointUri, PrimaryKey);
        private static Database database = cosmosClient.GetDatabase("Orders");
        private static Container container = database.GetContainer("container_orders");

        [FunctionName("OrderEngine")]
        public static async Task Run([ServiceBusTrigger("placed", Connection = "AzureWebJobsServiceBus")]string myQueueItem, ILogger log)
        {
            log.LogInformation($"C# ServiceBus queue trigger function processed message: {myQueueItem}");

            // Parse the queue item as your data model
            var order = JsonConvert.DeserializeObject<Order>(myQueueItem);
            if (order != null)
            {
                await InsertOrder(order);
            }
        }

        private static async Task InsertOrder(Order order)
        {
            try
            {
                ItemResponse<Order> orderResponse = await container.CreateItemAsync<Order>(order, new PartitionKey(order.Id));
            }
            catch (CosmosException ex) when (ex.StatusCode == System.Net.HttpStatusCode.Conflict)
            {
                // A conflict means that an order with the same id already exists. Handle appropriately.
            }
        }
    }

    public class Order
    {
        [JsonProperty(PropertyName = "id")]
        public string Id { get; set; }

        [JsonProperty(PropertyName = "restaurantId")]
        public string RestaurantId { get; set; }

        [JsonProperty(PropertyName = "sourceId")]
        public string SourceId { get; set; }

        [JsonProperty(PropertyName = "storeName")]
        public string StoreName { get; set; }

        [JsonProperty(PropertyName = "userName")]
        public string UserName { get; set; }

        [JsonProperty(PropertyName = "observation")]
        public string Observation { get; set; }

        [JsonProperty(PropertyName = "timeReady")]
        public string TimeReady { get; set; }
    }
}

```

<!-- CONTACT -->
## Contact

- [Johnny A. Gutierrez R](linkedin.com/in/alexander-gutierrez-1016) - 4dagio.01@gmail.com
