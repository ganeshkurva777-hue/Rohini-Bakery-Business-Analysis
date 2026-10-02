# Rohini-Bakery-Business-Analysis
PostgreSQL database architecture and Power BI operational dashboards tracking economic profit margins, owner labor opportunity costs, and a 2-year predictive forecast.
# 🏪 Business Scenario
Rohini is a solo baker who founded her boutique enterprise in 2019, specializing in bespoke, premium custom cakes. To evaluate her operational efficiency, she meticulously recorded her business metrics from 2021 through late 2026, capturing transaction orders, her personal working hours, and monthly operational expenses.
• Data Grain & Privacy: The dataset is structured such that each individual cake represents a separate order record. Customers are uniquely identified via a combination of their name and mobile number. To maintain strict data privacy, all Personally Identifiable Information (PII) has been masked using synthetic customer_id keys, and each order is mapped to a unique order_id.
# 🎯 Core Analytical Objectives
Rohini requires data-driven evidence to resolve three primary strategic questions regarding her enterprise trajectory:
1. Future Trajectory: Based on historical run-rates, what path will the business take over a 2-year forward-looking strategic horizon?
2. Economic Profitability: Is the business genuinely profitable once accounting for operational overhead and the opportunity cost of skilled labor?
3. Business Continuation: What empirical evidence does the data provide to justify continuing or scaling the business operations?
