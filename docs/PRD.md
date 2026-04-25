# **JDER**

Product Requirements Document

Version 1.0 | Confidential

| **Product** | Jder - Indian Job Search & Aggregator Platform |
| ----------- | ---------------------------------------------- |

| **Version** | 1.0 |
| ----------- | --- |

| **Status** | Draft |
| ---------- | ----- |

| **Phase** | Phase 1 - Aggregator |
| --------- | -------------------- |

# **Table of Contents**

# **1\. Introduction**

## **1.1 Purpose**

This document defines the product requirements for Jder, an India-specific job search and aggregator platform. It outlines the problem, goals, user needs, features, and success metrics for Phase 1.

## **1.2 Problem Statement**

The Indian job market is dominated by platforms with poor search quality, outdated recommendation logic, and weak Tier 2/3 city support. Specifically:

- Naukri's search is keyword-matching with no semantic understanding
- Salary data is inconsistently formatted across listings
- Skills like 'React.js', 'ReactJS', and 'React' are treated as different entities
- Tier 2/3 city filtering is unreliable or absent
- Recommendations are not personalised - they are popularity-based at best

## **1.3 Product Vision**

Jder is a job search platform that is genuinely smarter than existing Indian job boards. Phase 1 aggregates listings from major Indian sources and delivers significantly better search and personalised recommendations. Phase 2 opens the platform to direct company postings.

## **1.4 Scope**

Phase 1 (this document): Aggregator. Scrape, normalise, index, search, recommend.

Phase 2 (future): Marketplace. Company postings, premium features, monetisation.

# **2\. Goals and Success Metrics**

## **2.1 Business Goals**

- Reach 1000 active users within 4 to 6 months of launch
- Establish Jder as the go-to job search tool for Indian tech professionals
- Build enough user trust and data to justify Phase 2 (company postings)

## **2.2 Success Metrics**

| **Metric**                   | **Target**    | **Timeframe**          |
| ---------------------------- | ------------- | ---------------------- |
| Active users                 | 1,000         | 4-6 months post-launch |
| Week-over-week return rate   | \> 30%        | Ongoing                |
| Search-to-click rate         | \> 25%        | First 30 days          |
| Recommendation click-through | \> 15%        | After 500 users        |
| Job listing freshness        | < 6 hours old | Ongoing                |
| Search latency (p95)         | < 300ms       | At launch              |

# **3\. Users and Personas**

## **3.1 Primary Users**

### **The Job Seeker**

- Tech professional, 1-8 years experience
- Based in Tier 1 city (Bangalore, Hyderabad, Pune, Mumbai, Delhi NCR) or Tier 2/3 city
- Frustrated with irrelevant search results on Naukri
- Wants salary transparency and skill-matched results

### **The Fresher**

- Final year engineering student or recent graduate
- Looking for entry-level roles or internships
- Primarily uses Internshala or LinkedIn today
- Price-sensitive - free product only

## **3.2 Out of Scope (Phase 1)**

- Recruiters and HR teams (Phase 2)
- Non-tech roles (add later based on demand)
- International job seekers

# **4\. Features**

## **4.1 Core Features - Phase 1**

| **Feature**                  | **Description**                                                                                            | **Priority** |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------ |
| Job Search                   | Full-text search across title, skills, description with filters for location, salary, experience, job type | P0           |
| Geo Search                   | Search by city name or radius. All Indian cities geocoded. Tier 2/3 supported.                             | P0           |
| Skill Normalisation          | React.js = ReactJS = React. Canonical skill mapping.                                                       | P0           |
| Salary Normalisation         | All salary formats converted to annual INR range.                                                          | P0           |
| User Auth                    | Email/password registration and login. JWT-based.                                                          | P0           |
| Saved Jobs                   | Save jobs to a personal list. Persisted across sessions.                                                   | P1           |
| Personalised Recommendations | Job recommendations based on user profile and behaviour.                                                   | P1           |
| Job Alerts                   | Email alerts when new jobs match saved criteria.                                                           | P1           |
| Search Facets                | Filter sidebar: city, job type, salary range, experience level.                                            | P1           |
| Application Tracker          | Track application status per job (applied, interviewing, offer).                                           | P2           |

## **4.2 Data Sources**

- Naukri.com - highest volume, Tier 1 and 2 coverage
- Foundit (ex-Monster India) - mid-market coverage
- Internshala - freshers and internships
- Wellfound - Indian startup roles (Phase 2)
- LinkedIn India - Phase 2 due to scraping complexity

## **4.3 Non-Functional Requirements**

| **Requirement**              | **Target**                               |
| ---------------------------- | ---------------------------------------- |
| Search latency (p95)         | < 300ms                                  |
| Recommendation latency (p95) | < 300ms (warm cache)                     |
| Listing freshness            | < 6 hours from crawl to searchable       |
| Uptime                       | \> 99.5%                                 |
| Mobile responsiveness        | Full support - Next.js responsive layout |
| Data accuracy                | \> 95% salary normalisation accuracy     |

# **5\. Constraints and Assumptions**

## **5.1 Constraints**

- No paid advertising budget in Phase 1 - growth is organic only
- Two-person team maximum in Phase 1 - scope must stay lean
- Scraping is the only data source - no official API agreements in Phase 1
- India-only - no international listings in Phase 1

## **5.2 Assumptions**

- Naukri, Foundit, and Internshala scraping is technically feasible with proxy rotation
- Elasticsearch geo queries perform acceptably for Indian city data
- 50+ waitlist signups before launch confirms enough initial interest
- Users will tolerate up to 6 hours of listing latency from crawl to search

# **6\. Roadmap**

| **Phase**       | **Timeline** | **Key Deliverables**                                            |
| --------------- | ------------ | --------------------------------------------------------------- |
| 0 - Validate    | Week 0-1     | Landing page live. 50+ email signups collected.                 |
| 1A - Skeleton   | Week 1-4     | Naukri crawler working. Basic ES search. Next.js UI.            |
| 1B - Ship       | Week 4-8     | All 3 crawlers live. Auth. Saved jobs. Recommendations. Launch. |
| 1C - Grow       | Month 2-6    | Job alerts. Iterate on search quality. Hit 1000 users.          |
| 2 - Marketplace | Month 6+     | Company postings. Premium tier. B2B outreach.                   |

# **7\. Out of Scope - Phase 1**

- Company job postings (Phase 2)
- Resume upload or parsing
- In-platform messaging between candidates and employers
- Mobile native apps (iOS/Android)
- Paid features or subscriptions
- Analytics dashboard for job market trends
- ATS integrations (Lever, Greenhouse, Keka)
