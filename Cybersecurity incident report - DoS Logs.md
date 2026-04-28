# Cybersecurity Incident Report:

| Indicators that led to discovery of incident |
| :--- |
| Connection issues were report by multiple employees trying to connect to the company website. A ticket was placed to research the connection issue |

| Investigation of incident |
| :---- | 
| Today there was a report that there were connection issues with employees to the company website. There does appear to be an abnormal amount of requests to the website from an individual IP, 203.0.113.0. <br><br> The logs show that: In the beginning, there were only a few attempts by the IP, but as time went on, the attempts increased. As the attempts increased, the logs started to show timeout and reset acknowledgment errors due to the overload of the server. <br><br> Indicators suggest a Denial of Service (DoS) attack on the server by an individual actor, using a specific technique called SYN flooding. |

| Explanation of a Denial of Service (DoS) |
| :---- |
| When website visitors try to establish a connection with the web server, a three-way handshake occurs using the TCP protocol. The three steps of the handshake are as follows: <br><br> 1.  The first step of the TCP protocol is a synchronized (SYN) request from a user to the web server. <br><br> 2. The web server will verify and acknowledge the request, allowing for a connection to occur with a synchronized, acknowledgement request (SYN, ACK) sent to the requesting user device. <br><br> 3. The user device will then receive this request, resulting in a acknowledgement (ACK) being sent to the web server, resulting in a connection. <br><br>  With a Denial of Service (DoS) attack, a single actor can send a large number of SYN requests to the web server, resulting in the flooding of the server with these SYN requests. If the server runs out of resources to handle the normal SYN requests in addition to the abnormally large SYN requests from the bad actor, errors will occur. These errors can be timeout errors, like a reset acknowledgement error (RST, ACK), not allowing a user to retrieve the website. <br><br> Logs indicate a single, possible threat actor, with the IP address of 203.0.113.0, requesting an abnormal amount of SYN requests. Which depleted the resources of the web server, causing website timeouts and acknowledgement errors, preventing connections to the website. |
