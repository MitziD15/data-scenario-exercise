1. First response time by team Write a query returning the average first_response_minutes for tickets closed in the last 30 days, grouped by agent team.

```sql
SELECT agents.team, AVG(tickets.first_response_minutes)
FROM tickets
JOIN agents ON tickets.agent_id = agents.agent_id
WHERE tickets.status= "closed"
AND tickets.closed_at>= CURRENT_DATE - INTERVAL "30 days"
GROUP by agents.team;
```


2. Agents with above-average reopen rates Write a query returning each agent's reopen rate (reopened tickets ÷ total tickets they handled) for agents whose rate is higher than their own team's average.
```sql
SELECT agent_id, team, reopened_tickets^1.0/total_tickets AS reopen_rate
INTO agent_reopen_rates
FROM(
      SELECT tickets.agent_id, agents.team,
          COUNT(*) AS total_tickets,
          SUM(CASE WHEN tickets.reopened_count>0 THEN 1 ELSE 0 END) AS reopened_tickets
      FROM tickets
      JOIN agents ON tickets.agent_id = agents.agent_id
      GROUP BY tickets.agent_id, agents.team
) AS summary_per_agent

SELECT agent_reopen_rates.agent_id, agent_reopen_rates.team, agent_reopen_rates.reopen_rate
FROM agent_reopen_rates
JOIN(
    SELECT team, AVG(reopen_rate) AS average_team
    FROM agent_reopen_rates
    GROUP BY team
) AS average ON agent_reopen_rates.team = average_team
WHERE agent_reopen_rates.reopen_rate=average.average_team;

```

3. CSAT trend by category Write a query returning the average CSAT score per category, per month, for the last 3 months.

4. Digging in Beyond the three tables above, what additional data would you want to pull to understand the Billing drop — and which of the existing tables would you start with, and why?

5. Testing a theory Pick one specific theory for what might be driving the drop. State your theory, then write a query using the tables above that would help confirm or rule it out.

6. What you'd actually do Say your query in Question 5 showed that reopened tickets in Billing spiked, and most of the reopens trace back to one specific issue type (e.g., refund timing questions). What would you actually do with that in the next week? Be concrete. What would you say to the team, what (if anything) would you change in a process or macro, and how would you know if it worked?

7. Reporting up and coaching down You need to update your own manager on this in two sentences, and separately coach one agent on it in a 1:1. How would those two conversations differ?

