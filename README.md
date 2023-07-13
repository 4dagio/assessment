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

* [[Node]](https://nodejs.org/en/about)

## High-level architecture

![image](https://github.com/4dagio/assessment/assets/3275936/faf1a616-440e-468f-a0ba-fabf34423692)

### Notes: it is assumed that:


1. The data source has the capacity to consume rest or soap services.
2. The amount of transactions are within the capacity of the premiere layer of the azure api management (approx. 100,000,000 calls each month).
3. Notifications will be sent to mobile devices.
4. Observability will be enabled with AppInsights and Monitor.

### Explanation



### NFR

<!-- GETTING STARTED -->
## Codebase



<!-- CONTACT -->
## Contact

Your Name - [@twitter_handle](https://twitter.com/twitter_handle) - email@email_client.com

Project Link: [https://github.com/github_username/repo_name](https://github.com/github_username/repo_name)

<p align="right">(<a href="#readme-top">back to top</a>)</p>



