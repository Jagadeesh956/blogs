- ==What does Major Incidents tell us about knowledge gaps ?==
- ==Why do incidents response or resolution time matters for production ?==**

   
**A maintenance symptom kept continued resulting in huge impact to Application**

*A Java application running on multiple VMs started throwing errors as "cannot UPSERT on a read only DB" , from Postgres server.  In simple terms , an application is trying to send a DB request to a server which is not a primary that can accepts writes out of a 3 node DB cluster.* 

**15min later when the alerts continues  to slack with more than 10K failures at 5 min interval....!** 

Support team opened a bridge , page Database Operations to get the reason for sudden error , resolution needed to solve it . 


DB team validates health of the cluster , confirms everything looking good and no changes at database end . Application team asks multiple questions "what causes this error", "why it is happening without any change" and "why it continues to happen" so on....

* Application team agrees that same errors were observed many times during primary switch , resolves automatically in sometime .  

DB team confirms open connections in the secondary/remote server of the cluster which should be the reason for the error . Requested by application team , DB team tries to clear the connections on remote server which keeps piling up again ..

The question still remains unanswered **" Why all of a sudden application is trying to connect to remote node of the cluster , not active master "**

**1 hour later** 
Debugging continues , more teams on to the call including LoadBalancer , Infrastructure Operations and downstreams that are impacted .... 

Loadbalancer team identifies single pool member of the VIP being active since many days while the other 2 nodes are inactive . An error for health check was found even from the single active node from the loadbalancer logs , recovered after next polling . People on the call raises questions against the behaviour , slack updates goes as "Pool members of VIP down for DB clusters" impacting multiple application servers doing DB updates.

DB team gets paged for similar bridges reporting similar DB errors that are impacts other apps and confirms BAU after sometime . No action from any team , apps are back BAU ... 

**An Enterprise level Issue for DB servers has been confirmed  with no root cause ** 

50-100 people on call, having cross questions to different teams , validating various things but nothing really concludes.... 

**2 hours since the start of the issue ...Application is still bleeding with same errors crossing 3M Errors** 

DB team manually reviews logs at server level , matches the timeframe when issue started to an observation stating "Postgres process shutting down" log  on master nodes at same time . Issue concluded as "something happened at server level to bring down database node" which exactly matches to when health check from LoadBalancer failed . 

Someone from Engineering team suggests to restart an instance of impacted application to see if the failures stop from that instance .... ==**Boom, it works perfectly fine after restart**== 

==**..................A resolution is found ... just restart the impacted application.................**== 


**3 hours from the beginning** 

Everyone takes some breath , application servers starts coming back to BAU . Leaderships gets updates as we are BAU from application end .. But , question of root cause still remains unanswered.... debugging on what caused this still continues from multiple teams, application support wants root cause from DB teams .. 

DB team brings in their SMEs onto call, many questions remains unanswered ... 

1) What happened when master node went down and came back suddenly ?
2) Why primary switch didn't happen when an active master went down ?
3) How application started trying to connect with secondary server while no master switch ?
4) Why applications still trying to connect to secondary nodes even after connections cleanup .


***The SME from DB team confirms when health check request from LoadBalancer fails , the DNS lookup for the cluster VIP goes round-robin across all the nodes of the cluster .*** 

That answers why application nodes started connecting to some random server and started throwing **"cannot UPSERT on a read only DB"**

**Followup Questions in my mind :-** 
* **Why application continue to throw error even after LoadBalancer health check succeded ?**
  **Answer:-**  Application process [ JVM ] never tried to do new DNS query as the active TCP connection exists with a DB host . DNS caching eliminated new lookup on DB VIP until restarted . 
* **How did LoadBalancer face health check failure ?** 
  **Answer:-** Postgres servers including primary , near-standby and far-standby had a sudden restart in a span of few seconds which automatically recovered , but none of them available to serve health check request .
* **Why leader switch didn't happen if primary server of Postgres Cluster went down ?**
  **Answer:-** Patroni status check enabled with 30sec of interval to avoid split-brain  **both nodes of a DB cosidering themself as leaders at same time** status of cluster nodes .


The entire analysis of this issue took around 4 hours of time from various teams with multiple arguments between teams protecting themself from no issue at their end while technical glitch resides at multiple places to avoid this situation from happening again . 


**MY TECHNICAL VIEW** 

 * A JVM process kept connecting to the same backend server without doing active DNS lookup as an active TCP connection exists , resulted in continuous failures at application layer . 
 * Patroni status check timegap [ 30sec ] doesn't match with LoadBalancer Health Check frequency [ 5 sec ] which caused  a round robin DNS results impacting DB VIPs. 
 * Knowledge gap from various teams resulted in 4hours of MTTR which would have been less than 15min with a simple restart of impacted services . 
   
   
   





