# Http

agent.ts

We create an http agent with better abilities with agentkeepalive one http one https. W eupdate https metrics with updateHttpAgentMetrics using prometheus client. with watch agent we log metrics every 5 sec. In gathering stats we caculate free socket, busy sockets and requetss of a certain type with errors and give them to prometeaus.