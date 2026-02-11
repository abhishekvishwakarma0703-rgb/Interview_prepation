# Rate Limiter - Complete System Design Guide

**A Comprehensive Reference for Understanding and Designing Rate Limiting Systems**

-----

## Table of Contents

1. [Introduction & Motivation](#1-introduction--motivation)
1. [The Core Problem](#2-the-core-problem)
1. [Requirements & Scope](#3-requirements--scope)
1. [Rate Limiting Algorithms (Deep Dive)](#4-rate-limiting-algorithms)
- Fixed Window Counter
- Sliding Window Log
- Sliding Window Counter
- Token Bucket
- Leaky Bucket
1. [Distributed Rate Limiting](#5-distributed-rate-limiting)
1. [System Design & Architecture](#6-system-design--architecture)
1. [Advanced Topics](#7-advanced-topics)
1. [Real-World Case Studies](#8-real-world-case-studies)
1. [Interview Framework](#9-interview-framework)

-----

## 1. Introduction & Motivation

### 1.1 What is Rate Limiting?

**Rate Limiting** is a technique to control the rate at which users or services can access a resource or perform an action.

```
Without Rate Limiting:
User sends 1,000,000 requests in 1 second
         ↓
    All hit your API
         ↓
    Server overloaded → Crashes 💥

With Rate Limiting:
User sends 1,000,000 requests in 1 second
         ↓
    First 100 allowed → rest rejected
         ↓
    Server stays healthy ✓
    User gets: "429 Too Many Requests"
```

### 1.2 Why Do We Need Rate Limiting?

**Problem 1: Resource Protection**

```
Scenario: Your API server can handle 10,000 QPS maximum

Without rate limiting:
├─ One user sends 50,000 requests/second
├─ Server CPU/memory maxed out
├─ ALL users experience slow response
└─ System becomes unavailable

With rate limiting:
├─ Limit each user to 100 requests/second
├─ Abusive user gets throttled
├─ Other users continue to work normally
└─ System remains stable
```

**Problem 2: Preventing Abuse & Attacks**

```
DDoS Attack Scenario:
├─ Attacker controls 10,000 bots
├─ Each bot sends 100 requests/second
├─ Total: 1,000,000 requests/second
└─ Your servers: 💀

Rate Limiting Defense:
├─ Limit per IP: 10 requests/second
├─ 10,000 IPs × 10 req/sec = 100,000 req/sec
├─ Still high, but manageable
└─ Combined with other defenses (IP blocking, CAPTCHA)
```

**Problem 3: Cost Control**

```
Third-party API costs:
├─ Google Maps API: $5 per 1,000 requests
├─ Without rate limiting: User bug causes 10M requests
├─ Cost: $50,000 in one day! 💸
└─ With rate limiting: Capped at $500/day ✓

Database query costs:
├─ Complex query costs 100ms CPU time
├─ Infinite loop causes 10,000 queries
├─ Database overloaded
└─ Rate limit prevents runaway queries
```

**Problem 4: Fair Resource Distribution**

```
Shared SaaS Platform:
├─ 1000 customers share infrastructure
├─ Customer A: 1,000 req/sec
├─ Customer B: 100,000 req/sec (monopolizes resources)
└─ Customer A gets slow service (unfair!)

With rate limiting per customer:
├─ Each customer: 1,000 req/sec maximum
├─ Fair usage across all customers
└─ Everyone gets predictable performance
```

### 1.3 Real-World Examples

|Service        |Rate Limit             |Why                                      |
|---------------|-----------------------|-----------------------------------------|
|**Twitter API**|300 requests / 15 min  |Prevent scraping, ensure fair access     |
|**GitHub API** |5,000 requests / hour  |Control server load, prevent abuse       |
|**Stripe API** |100 requests / second  |Protect payment processing infrastructure|
|**Google Maps**|$200 free credit/month |Cost control, prevent bill shock         |
|**OpenAI API** |Tier-based (tokens/min)|GPU resource management                  |

-----

## 2. The Core Problem

### 2.1 The Challenge

**Question:** How do you enforce “100 requests per minute” for millions of users?

**Challenges:**

```
Challenge 1: Counting Requests
├─ How do you count requests accurately?
├─ Where do you store the count?
├─ How do you handle distributed servers?
└─ What happens when counter resets?

Challenge 2: Performance
├─ Rate limit check must be FAST (< 1ms)
├─ Cannot slow down every request
├─ Must scale to millions of users
└─ Must handle high concurrent load

Challenge 3: Accuracy
├─ Must be fair (no user gets unfair advantage)
├─ Must prevent gaming the system
├─ Must handle edge cases (burst traffic)
└─ Must be consistent across multiple servers

Challenge 4: Resource Efficiency
├─ Cannot use too much memory
├─ Cannot use too much CPU
├─ Cannot use too much network bandwidth
└─ Must clean up old data
```

### 2.2 Simple (Naive) Approach - Why It Fails

**Attempt 1: In-Memory Counter**

```python
# Store in application memory
user_requests = {}  # user_id -> count

def is_allowed(user_id):
    if user_id not in user_requests:
        user_requests[user_id] = 0
    
    user_requests[user_id] += 1
    
    if user_requests[user_id] <= 100:
        return True
    return False
```

**Problems:**

```
❌ When does the counter reset? Never!
❌ What if you have multiple servers?
   ├─ Server 1 has count = 50
   ├─ Server 2 has count = 50
   └─ User made 100 requests total, but both allow more!
❌ Memory leak - user_requests grows forever
❌ Lost on server restart
```

**Attempt 2: Add Time Window**

```python
import time

user_requests = {}  # user_id -> {'count': 0, 'window_start': timestamp}

def is_allowed(user_id):
    now = time.time()
    
    if user_id not in user_requests:
        user_requests[user_id] = {'count': 1, 'window_start': now}
        return True
    
    # Check if window expired (60 seconds)
    if now - user_requests[user_id]['window_start'] >= 60:
        # Reset window
        user_requests[user_id] = {'count': 1, 'window_start': now}
        return True
    
    # Within window
    user_requests[user_id]['count'] += 1
    
    if user_requests[user_id]['count'] <= 100:
        return True
    return False
```

**Better, but still problems:**

```
❌ Still doesn't work with multiple servers
❌ Edge case: Burst at window boundary
   ├─ 00:00:59 - User sends 100 requests (allowed)
   ├─ 00:01:00 - Window resets
   ├─ 00:01:01 - User sends 100 requests (allowed)
   └─ Result: 200 requests in 2 seconds! (2x the limit)
❌ Memory still grows unbounded
```

**This is why we need sophisticated algorithms!**

-----

## 3. Requirements & Scope

### 3.1 Functional Requirements

```
Core Operations:
├─ allow_request(user_id) → boolean
├─ Configure rate limit rules (100/min, 1000/hour, etc.)
├─ Support multiple rate limit dimensions:
│  ├─ Per user
│  ├─ Per IP address
│  ├─ Per API key
│  └─ Per endpoint
└─ Return meaningful error messages

Advanced (optional):
├─ Different limits for different user tiers (free vs premium)
├─ Dynamic rate limits based on server load
├─ Temporary rate limit increases
└─ Rate limit analytics/monitoring
```

### 3.2 Non-Functional Requirements

|Requirement          |Target              |Why                                               |
|---------------------|--------------------|--------------------------------------------------|
|**Latency**          |< 1ms (p99)         |Cannot slow down API requests                     |
|**Accuracy**         |99.9%+              |Fair enforcement, prevent gaming                  |
|**Scalability**      |Handle 10M users    |Large-scale systems                               |
|**Availability**     |99.99%              |Rate limiter failure shouldn’t break entire system|
|**Memory Efficiency**|O(users) space      |Cannot store unlimited data                       |
|**Fault Tolerance**  |Graceful degradation|Fail open or fail closed?                         |

### 3.3 Failure Modes

**Question: When rate limiter fails, what should happen?**

```
Option A: Fail Open (Allow all requests)
├─ Pro: Service remains available
├─ Con: No protection during outage
└─ Use when: Availability > Security (e.g., social media)

Option B: Fail Closed (Reject all requests)
├─ Pro: Protection maintained
├─ Con: Service unavailable
└─ Use when: Security > Availability (e.g., banking)

Option C: Hybrid
├─ Allow requests with high priority/authentication
├─ Reject anonymous/low-priority requests
└─ Use when: Need balance (e.g., e-commerce)
```

-----

## 4. Rate Limiting Algorithms

### 4.1 Fixed Window Counter

**Concept:** Divide time into fixed windows (e.g., 1 minute). Count requests in each window.

**How It Works:**

```
Time windows (60 seconds each):
┌──────────────┬──────────────┬──────────────┐
│ Window 1     │ Window 2     │ Window 3     │
│ 00:00 - 01:00│ 01:00 - 02:00│ 02:00 - 03:00│
└──────────────┴──────────────┴──────────────┘

Limit: 100 requests per minute

Window 1 (00:00 - 01:00):
├─ Request 1 at 00:00:10 → count = 1 → Allow ✓
├─ Request 2 at 00:00:20 → count = 2 → Allow ✓
├─ ...
├─ Request 100 at 00:00:50 → count = 100 → Allow ✓
└─ Request 101 at 00:00:55 → count = 101 → Reject ✗

Window 2 (01:00 - 02:00):
└─ Counter resets to 0 at exactly 01:00:00
```

**Implementation:**

```python
import time
import math

class FixedWindowCounter:
    def __init__(self, max_requests, window_size_seconds):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.counters = {}  # user_id -> {'count': int, 'window_start': timestamp}
    
    def allow_request(self, user_id):
        now = time.time()
        
        # Calculate current window start
        current_window = math.floor(now / self.window_size) * self.window_size
        
        if user_id not in self.counters:
            self.counters[user_id] = {
                'count': 1,
                'window_start': current_window
            }
            return True
        
        user_data = self.counters[user_id]
        
        # Check if we're in a new window
        if user_data['window_start'] < current_window:
            # Reset for new window
            self.counters[user_id] = {
                'count': 1,
                'window_start': current_window
            }
            return True
        
        # Same window - increment counter
        if user_data['count'] < self.max_requests:
            user_data['count'] += 1
            return True
        
        return False  # Exceeded limit

# Usage
limiter = FixedWindowCounter(max_requests=100, window_size_seconds=60)

# Simulate requests
for i in range(150):
    user_id = "user_123"
    allowed = limiter.allow_request(user_id)
    print(f"Request {i+1}: {'Allowed' if allowed else 'Rejected'}")
```

**Visual Example:**

```
Limit: 5 requests per minute

Timeline:
├─ 00:00:10 - Request 1 → count = 1 → ✓ Allowed
├─ 00:00:20 - Request 2 → count = 2 → ✓ Allowed
├─ 00:00:30 - Request 3 → count = 3 → ✓ Allowed
├─ 00:00:40 - Request 4 → count = 4 → ✓ Allowed
├─ 00:00:50 - Request 5 → count = 5 → ✓ Allowed
├─ 00:00:55 - Request 6 → count = 6 → ✗ Rejected (limit reached)
├─ 01:00:00 - Window resets → count = 0
└─ 01:00:05 - Request 7 → count = 1 → ✓ Allowed
```

**The Boundary Problem (Critical!):**

```
Limit: 100 requests per minute

Timeline:
├─ 00:00:00 - Window 1 starts
├─ 00:00:40 - User sends 50 requests → Allowed ✓
├─ 00:00:59 - User sends 50 requests → Allowed ✓ (total: 100)
│   [End of Window 1: 100 requests in last 60 seconds]
│
├─ 01:00:00 - Window 2 starts, counter resets to 0
├─ 01:00:01 - User sends 50 requests → Allowed ✓
├─ 01:00:20 - User sends 50 requests → Allowed ✓ (total: 100)
│
└─ Analysis:
    From 00:00:40 to 01:00:20 (40 seconds):
    └─ User sent 200 requests!
    └─ That's 300 requests per minute rate!
    └─ DOUBLE the intended limit! ✗
```

**Visualization of the Problem:**

```
Window 1: 00:00 - 01:00          Window 2: 01:00 - 02:00
┌────────────────────────────┐   ┌────────────────────────────┐
│ ................ ██████████│   │██████████ ................│
│                  50 req    │   │50 req                      │
│                  at :40-:59│   │at :00-:20                  │
└────────────────────────────┘   └────────────────────────────┘
                     └──────────┬──────────┘
                                │
                        40 seconds: 200 requests!
                        (Should be max 67 for 40 seconds)
```

**Pros:**

```
✓ Simple to implement
✓ Memory efficient: O(1) per user
✓ Fast: O(1) lookup and update
✓ Easy to understand
```

**Cons:**

```
✗ Burst traffic at window boundaries (can exceed limit 2x)
✗ Hard reset creates usage spikes
✗ Not fair across time
```

**When to Use:**

```
✓ Simple use cases with relaxed accuracy
✓ Internal services (not customer-facing)
✓ When implementation simplicity matters most
✗ Don't use when strict rate limiting needed
```

-----

### 4.2 Sliding Window Log

**Concept:** Keep a log (list) of timestamps for each request. Count requests in the last N seconds.

**How It Works:**

```
Limit: 5 requests per minute (60 seconds)

User's request log (stored as timestamps):
[00:00:10, 00:00:25, 00:00:40, 00:01:05, 00:01:20]

At time 00:01:30, new request arrives:
1. Remove timestamps older than 60 seconds
   └─ Current time: 00:01:30 (90 seconds)
   └─ Remove: 00:00:10 (80 seconds ago - too old)
   └─ Remove: 00:00:25 (65 seconds ago - too old)
   └─ Keep: 00:00:40, 00:01:05, 00:01:20

2. Count remaining timestamps: 3 requests

3. Check: 3 < 5? Yes → Allow request ✓

4. Add current timestamp to log:
   [00:00:40, 00:01:05, 00:01:20, 00:01:30]
```

**Implementation:**

```python
import time
from collections import deque

class SlidingWindowLog:
    def __init__(self, max_requests, window_size_seconds):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        # user_id -> deque of timestamps
        self.logs = {}
    
    def allow_request(self, user_id):
        now = time.time()
        
        if user_id not in self.logs:
            self.logs[user_id] = deque()
        
        user_log = self.logs[user_id]
        
        # Remove timestamps outside the window
        cutoff_time = now - self.window_size
        while user_log and user_log[0] <= cutoff_time:
            user_log.popleft()
        
        # Check if we can allow this request
        if len(user_log) < self.max_requests:
            user_log.append(now)
            return True
        
        return False
    
    def cleanup(self):
        """Optional: Remove empty user logs to save memory"""
        now = time.time()
        cutoff = now - self.window_size
        
        to_delete = []
        for user_id, log in self.logs.items():
            # Remove old entries
            while log and log[0] <= cutoff:
                log.popleft()
            
            # Mark empty logs for deletion
            if not log:
                to_delete.append(user_id)
        
        for user_id in to_delete:
            del self.logs[user_id]

# Usage
limiter = SlidingWindowLog(max_requests=5, window_size_seconds=60)

# Test
print(limiter.allow_request("user_123"))  # True
time.sleep(0.1)
print(limiter.allow_request("user_123"))  # True
# ... 3 more requests ...
print(limiter.allow_request("user_123"))  # True (5th)
print(limiter.allow_request("user_123"))  # False (exceeds limit)
```

**Visual Example:**

```
Limit: 5 requests per 60 seconds

Timeline with sliding window:

Time: 00:00:00
Log: []
Request → count = 0 < 5 → ✓ Allowed
Log: [00:00:00]

Time: 00:00:15
Log: [00:00:00] (1 request in last 60s)
Request → count = 1 < 5 → ✓ Allowed
Log: [00:00:00, 00:00:15]

Time: 00:00:30
Log: [00:00:00, 00:00:15] (2 requests in last 60s)
Request → count = 2 < 5 → ✓ Allowed
Log: [00:00:00, 00:00:15, 00:00:30]

Time: 01:00:10
Window: [00:00:10 - 01:00:10]
Old log: [00:00:00, 00:00:15, 00:00:30]
After cleanup: [00:00:15, 00:00:30] (00:00:00 is older than 60s)
Request → count = 2 < 5 → ✓ Allowed
Log: [00:00:15, 00:00:30, 01:00:10]
```

**No Boundary Problem!**

```
Limit: 100 requests per minute

Timeline:
00:00:40 - User sends 50 requests
         - Log has 50 timestamps

00:00:59 - User sends 50 requests  
         - Log has 100 timestamps
         - Next request rejected ✗

01:00:01 - User tries to send more
         - Window: [00:00:01 - 01:00:01]
         - Cleanup: Remove timestamps before 00:00:01
         - Only timestamps from 00:00:40 onwards remain
         - Log still has ~100 requests
         - Request rejected ✗

01:00:40 - User tries again
         - Window: [00:00:40 - 01:00:40]
         - Cleanup: Remove timestamps before 00:00:40
         - First 50 requests (from 00:00:40) now removed
         - Only 50 timestamps remain (from 00:00:59)
         - Request allowed ✓

Result: Smooth, accurate rate limiting!
```

**Memory Analysis:**

```
Fixed Window: O(1) per user
└─ Only store: count + window_start

Sliding Window Log: O(limit) per user
└─ Store: All timestamps in window
└─ Example: 100 req/min → 100 timestamps per user
└─ 1M users × 100 timestamps × 8 bytes = 800 MB

For high-rate limits, this can be expensive!
```

**Pros:**

```
✓ Very accurate - no boundary problem
✓ True sliding window behavior
✓ Fair enforcement across time
```

**Cons:**

```
✗ Memory intensive: O(limit × users)
✗ Cleanup overhead (removing old timestamps)
✗ Can be slow with very high rate limits
```

**When to Use:**

```
✓ When accuracy is critical
✓ Low to medium rate limits (< 1000/window)
✓ Security-sensitive applications
✗ Don't use for very high rate limits (memory cost)
```

-----

### 4.3 Sliding Window Counter (Hybrid)

**Concept:** Combine fixed window efficiency with sliding window accuracy. Use weighted count from two adjacent windows.

**How It Works:**

```
Current time: 00:01:15 (75 seconds = 1 minute 15 seconds)
Window size: 60 seconds
Limit: 100 requests per minute

┌─────────────────────────┬─────────────────────────┐
│   Previous Window       │    Current Window       │
│   00:00 - 01:00        │    01:00 - 02:00       │
│   Count: 80 requests    │    Count: 30 requests   │
└─────────────────────────┴─────────────────────────┘
                          ↑
                    Current time: 01:15
                    (15 seconds into current window)

Calculate estimated count for last 60 seconds:

Time range we care about: [00:01:15 - 01:01:15]

Overlap with previous window: 45 seconds (00:01:15 - 01:00:00)
Overlap with current window: 15 seconds (01:00:00 - 01:01:15)

Formula:
Estimated count = (Previous window count × overlap% with previous)
                + (Current window count)

Estimated count = (80 × 45/60) + 30
                = (80 × 0.75) + 30
                = 60 + 30
                = 90 requests

90 < 100? Yes → Allow request ✓
```

**Detailed Visual:**

```
Timeline:
├─────────────────────────────────────────────────────────────┤
│                Previous Window         │   Current Window   │
│                00:00 - 01:00          │   01:00 - 02:00   │
│                (80 requests)           │   (30 requests)    │
├─────────────────────────────────────────────────────────────┤
                                          ↑
                                    Current: 01:15

Sliding window we want to check: [00:01:15 - 01:01:15]
├────────────────────────────────────┤
│ Part in prev window │ Part in curr  │
│   (45 seconds)      │   (15 sec)    │
│   75% of window     │   25% of win  │
├────────────────────────────────────┤

Weight calculation:
├─ Previous window contribution: 80 × 0.75 = 60
├─ Current window contribution: 30 × 1.0 = 30
└─ Total estimate: 90 requests
```

**Implementation:**

```python
import time
import math

class SlidingWindowCounter:
    def __init__(self, max_requests, window_size_seconds):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        # user_id -> {'prev_count': int, 'prev_window': timestamp,
        #             'curr_count': int, 'curr_window': timestamp}
        self.windows = {}
    
    def allow_request(self, user_id):
        now = time.time()
        
        # Calculate current window start
        current_window = math.floor(now / self.window_size) * self.window_size
        
        if user_id not in self.windows:
            self.windows[user_id] = {
                'prev_count': 0,
                'prev_window': current_window - self.window_size,
                'curr_count': 0,
                'curr_window': current_window
            }
        
        user_data = self.windows[user_id]
        
        # Check if we've moved to a new window
        if user_data['curr_window'] < current_window:
            # Shift windows: current becomes previous
            user_data['prev_count'] = user_data['curr_count']
            user_data['prev_window'] = user_data['curr_window']
            user_data['curr_count'] = 0
            user_data['curr_window'] = current_window
        
        # Calculate weighted count
        elapsed_in_current = now - user_data['curr_window']
        previous_window_weight = 1 - (elapsed_in_current / self.window_size)
        
        estimated_count = (
            user_data['prev_count'] * previous_window_weight +
            user_data['curr_count']
        )
        
        # Check if request allowed
        if estimated_count < self.max_requests:
            user_data['curr_count'] += 1
            return True
        
        return False

# Usage
limiter = SlidingWindowCounter(max_requests=100, window_size_seconds=60)

allowed = limiter.allow_request("user_123")
print(f"Request allowed: {allowed}")
```

**Example Walkthrough:**

```
Limit: 10 requests per minute

Scenario:
├─ 00:00:00 - 00:01:00: User makes 8 requests
├─ 00:01:00 - 00:02:00: User makes 4 requests so far
└─ Current time: 00:01:30 (30 seconds into second window)

Previous window: [00:00-00:01] = 8 requests
Current window: [00:01-00:02] = 4 requests
Time in current window: 30 seconds (50% of window)

Calculation:
├─ Previous window weight: 1 - (30/60) = 0.5
├─ Estimated count: (8 × 0.5) + 4 = 4 + 4 = 8
└─ 8 < 10 → Allow ✓

New request at 00:01:45 (45 seconds into window):
├─ Previous window weight: 1 - (45/60) = 0.25
├─ Current window now has 5 requests
├─ Estimated count: (8 × 0.25) + 5 = 2 + 5 = 7
└─ 7 < 10 → Allow ✓
```

**Accuracy Comparison:**

```
Scenario: Burst at boundary
├─ 00:00:50 - 00:01:00: 10 requests (in 10 seconds)
├─ 00:01:00 - 00:01:10: 10 requests (in 10 seconds)
└─ Total: 20 requests in 20 seconds (should be rejected!)

Fixed Window:
├─ Window 1 [00:00-01:00]: 10 requests → All allowed ✓
├─ Window 2 [01:00-02:00]: 10 requests → All allowed ✓
└─ Result: 20 requests allowed ✗ (WRONG - allows 2x limit)

Sliding Window Log:
├─ At 00:01:10, check last 60 seconds
├─ Finds 20 requests in last 60 seconds
└─ Result: Rejects requests after 10th ✓ (CORRECT)

Sliding Window Counter:
├─ At 00:01:05 (5 sec into window 2):
│  └─ Estimate: (10 × 0.917) + 5 = 14.17
│  └─ Rejects 5th request in window 2 ✓
└─ Result: Approximately correct (better than fixed)
```

**Memory Usage:**

```
Fixed Window: O(1) per user
├─ count + window_start

Sliding Window Log: O(limit) per user
├─ All timestamps

Sliding Window Counter: O(1) per user
├─ prev_count + curr_count + timestamps
└─ Only 2 counters per user!

Winner: Same memory as Fixed Window! ✓
```

**Pros:**

```
✓ Memory efficient: O(1) per user
✓ More accurate than fixed window
✓ Fast: O(1) lookup and update
✓ Smooth rate limiting (no hard resets)
```

**Cons:**

```
✗ Not perfectly accurate (estimation)
✗ Slightly more complex than fixed window
✗ Small inaccuracies at window boundaries
```

**When to Use:**

```
✓ Best balance of accuracy and efficiency
✓ Most production systems use this!
✓ High-scale applications (Twitter, Stripe)
✓ When you need good accuracy without memory overhead
```

-----

### 4.4 Token Bucket

**Concept:** Imagine a bucket that holds tokens. Tokens are added at a constant rate. Each request consumes one token.

**Analogy:**

```
Think of it like a vending machine with coupons:
├─ Bucket capacity: 100 coupons
├─ Refill rate: 10 coupons per second
├─ Each request costs 1 coupon
└─ If bucket empty → request denied

This allows bursts (use all 100 coupons quickly)
But sustained rate is limited (only 10/second long-term)
```

**How It Works:**

```
Bucket State:
┌─────────────────────────┐
│ Bucket Capacity: 100    │
│ Current Tokens: 75      │
│ Refill Rate: 10/second  │
└─────────────────────────┘

Timeline:
├─ 00:00:00 - Bucket has 75 tokens
├─ 00:00:01 - Refill: +10 tokens → 85 tokens
├─ 00:00:02 - Request arrives, consume 1 token → 84 tokens
├─ 00:00:03 - Refill: +10 tokens → 94 tokens
├─ 00:00:04 - 5 requests arrive, consume 5 tokens → 89 tokens
├─ 00:00:05 - Refill: +10 tokens → 99 tokens
├─ 00:00:06 - Burst: 50 requests → 49 tokens (all allowed!)
├─ 00:00:07 - Refill: +10 tokens → 59 tokens
└─ 00:00:08 - Burst: 100 requests → First 59 allowed, rest rejected
```

**Key Properties:**

```
1. Bucket Capacity (Burst Size):
   └─ Maximum tokens bucket can hold
   └─ Determines maximum burst

2. Refill Rate:
   └─ Tokens added per second
   └─ Determines sustained rate

Example:
├─ Capacity: 100 tokens
├─ Refill: 10 tokens/second
├─ Burst: Can handle 100 requests immediately
└─ Sustained: Max 10 requests/second over time
```

**Implementation:**

```python
import time

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        """
        capacity: Maximum tokens in bucket (burst size)
        refill_rate: Tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity  # Start with full bucket
        self.last_refill = time.time()
    
    def allow_request(self, tokens_needed=1):
        """
        Check if request can be allowed
        tokens_needed: Number of tokens required (default 1)
        """
        now = time.time()
        
        # Calculate tokens to add since last refill
        time_elapsed = now - self.last_refill
        tokens_to_add = time_elapsed * self.refill_rate
        
        # Refill bucket (but don't exceed capacity)
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
        
        # Check if we have enough tokens
        if self.tokens >= tokens_needed:
            self.tokens -= tokens_needed
            return True
        
        return False

# Usage
limiter = TokenBucket(capacity=100, refill_rate=10)

# Burst scenario
print("Burst test:")
for i in range(150):
    if limiter.allow_request():
        print(f"Request {i+1}: Allowed")
    else:
        print(f"Request {i+1}: Rejected")

# Sustained rate test
print("\nSustained rate test:")
for i in range(20):
    time.sleep(0.1)  # 100ms between requests
    if limiter.allow_request():
        print(f"Request {i+1}: Allowed")
    else:
        print(f"Request {i+1}: Rejected")
```

**Visual Representation:**

```
Token Bucket State Over Time:

Capacity: 10 tokens
Refill: 2 tokens/second

Time  Tokens  Event
────  ──────  ─────────────────────
0:00   10     Start (full bucket)
0:01   10     Refill +2, but capped at 10
0:02   10     Refill +2, but capped at 10
0:03   9      Request arrives (-1 token)
0:04   10     Refill +2, total 11 → capped at 10
0:05   5      Burst: 5 requests (-5 tokens)
0:06   7      Refill +2 (+2 tokens)
0:07   9      Refill +2 (+2 tokens)
0:08   3      Burst: 6 requests (only 9 available)
              First 9 allowed, 7th rejected
0:09   5      Refill +2 (+2 tokens)
0:10   7      Refill +2 (+2 tokens)

Graph:
Tokens
  10 │█████████
   9 │████████ █
   8 │████████
   7 │████████      █ █
   6 │████████
   5 │████████  █     █
   4 │████████
   3 │████████      █
   2 │████████
   1 │████████
   0 ├────────────────────────
     0  2  4  6  8  10 (seconds)
```

**Comparison: Token Bucket vs Fixed Window**

```
Scenario: Limit should be 10 requests/second
User sends 15 requests at once

Fixed Window (1-second windows):
├─ Window 1 [0:00-0:01]: First 10 allowed, 5 rejected
├─ Window 2 [0:01-0:02]: Counter resets, 10 more allowed
└─ Problem: Hard cutoff at window boundary

Token Bucket (capacity=10, refill=10/sec):
├─ Initial: 10 tokens available
├─ Burst: All 10 requests allowed immediately
├─ 5 requests rejected (no tokens left)
├─ After 0.5 seconds: +5 tokens refilled
├─ 5 more requests can be allowed
└─ Smooth: Allows burst, then gradual recovery
```

**Use Cases:**

```
Token Bucket is ideal for:

1. Network Traffic Shaping:
   └─ Allow bursts, but limit sustained rate
   └─ Example: ISP bandwidth limiting

2. API Rate Limiting with Bursts:
   ├─ Normal: 10 req/sec
   ├─ Allow occasional bursts up to 100 req
   └─ Example: Stripe API

3. Resource Allocation:
   └─ CPU time slices
   └─ I/O bandwidth allocation

Real Example - Stripe API:
├─ Bucket Capacity: 100 requests
├─ Refill Rate: 1 request/second
├─ Result: Can burst 100 requests immediately
└─ Then limited to 1 req/sec sustained
```

**Pros:**

```
✓ Allows controlled bursts
✓ Smooth rate limiting
✓ Memory efficient: O(1) per user
✓ Fast: O(1) per request
✓ Flexible (can tune capacity vs rate separately)
```

**Cons:**

```
✗ Slightly complex to understand
✗ Requires floating-point arithmetic
✗ Old tokens never expire (must handle edge cases)
```

**When to Use:**

```
✓ When bursts are acceptable (and even desirable)
✓ Network traffic shaping
✓ API rate limiting for flexibility
✓ When you want to be lenient to users
```

-----

### 4.5 Leaky Bucket

**Concept:** Imagine a bucket with a hole at the bottom. Requests are added to the bucket, and “leak out” at a constant rate.

**Analogy:**

```
Think of it like a funnel:
├─ Requests pour in at varying rates (top)
├─ Requests leak out at constant rate (bottom hole)
├─ If bucket overflows → request rejected
└─ Smooths out traffic spikes

Like a line at a theme park ride:
├─ People arrive in bursts
├─ Ride takes people at constant rate
└─ If line too long → people turned away
```

**How It Works:**

```
Bucket State:
┌─────────────────────────────────┐
│ Bucket Capacity: 10 requests    │
│ Current Queue: 3 requests       │
│ Leak Rate: 2 requests/second    │
└─────────────────────────────────┘

Every second: Process 2 requests from queue

Timeline:
├─ 00:00 - Queue: [R1, R2, R3]
├─ 00:01 - Leak 2 requests (R1, R2 processed)
│          Queue: [R3]
├─ 00:02 - 5 new requests arrive
│          Queue: [R3, R4, R5, R6, R7, R8]
├─ 00:03 - Leak 2 requests (R3, R4 processed)
│          Queue: [R5, R6, R7, R8]
├─ 00:04 - 10 new requests arrive
│          Queue size would be 14 (4 + 10)
│          But capacity is 10!
│          First 6 added, next 4 rejected ✗
└─ 00:05 - Leak 2 requests, queue: 8 remaining
```

**Key Difference from Token Bucket:**

```
Token Bucket:
├─ Tokens accumulate over time
├─ Can spend all tokens at once (burst)
└─ Refill continuously

Leaky Bucket:
├─ Requests accumulate in queue
├─ Processed at constant rate (no bursts)
└─ Leak continuously

Example:
├─ 100 requests arrive instantly
│
Token Bucket:
└─ If 100 tokens available → all processed immediately ✓

Leaky Bucket:
└─ All added to queue → processed at fixed rate over time
    (e.g., 10/second → takes 10 seconds to process all)
```

**Implementation (Queue-based):**

```python
import time
from collections import deque

class LeakyBucket:
    def __init__(self, capacity, leak_rate):
        """
        capacity: Maximum requests that can be queued
        leak_rate: Requests processed per second
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue = deque()
        self.last_leak = time.time()
    
    def _leak(self):
        """Process (leak) requests based on elapsed time"""
        now = time.time()
        elapsed = now - self.last_leak
        
        # Calculate how many requests to leak
        requests_to_leak = int(elapsed * self.leak_rate)
        
        if requests_to_leak > 0:
            # Remove leaked requests from queue
            for _ in range(min(requests_to_leak, len(self.queue))):
                self.queue.popleft()
            
            self.last_leak = now
    
    def allow_request(self):
        """Try to add request to bucket"""
        # First, leak existing requests
        self._leak()
        
        # Check if bucket has space
        if len(self.queue) < self.capacity:
            self.queue.append(time.time())
            return True
        
        return False  # Bucket full
    
    def get_queue_size(self):
        """Get current number of queued requests"""
        self._leak()
        return len(self.queue)

# Usage
limiter = LeakyBucket(capacity=10, leak_rate=2)

# Test burst
print("Burst of 15 requests:")
for i in range(15):
    if limiter.allow_request():
        print(f"Request {i+1}: Queued (queue size: {limiter.get_queue_size()})")
    else:
        print(f"Request {i+1}: Rejected (bucket full)")

# Wait and check queue drain
time.sleep(2)
print(f"\nAfter 2 seconds, queue size: {limiter.get_queue_size()}")
```

**Visual Representation:**

```
Leaky Bucket Over Time:

Capacity: 5
Leak Rate: 1 request/second

Time  Queue  Event
────  ─────  ──────────────────────
0:00  [R1]   Request 1 arrives
0:01  [R2]   R1 leaks, R2 arrives
0:02  [R2,R3,R4] R2 stays, R3,R4 arrive
0:03  [R3,R4,R5] R2 leaks, R5 arrives
0:04  [R4,R5]    R3 leaks
0:05  [R5,R6,R7,R8,R9] Burst arrives
0:06  [R6,R7,R8,R9,R10] R5 leaks, R10 arrives
                        (R11 would be rejected - full!)
0:07  [R7,R8,R9,R10]    R6 leaks
0:08  [R8,R9,R10]       R7 leaks

Queue Visualization:
Size
  5 │        ███████
  4 │      ███████████
  3 │    ██████████████
  2 │  ████████████████
  1 │██████████████████████
  0 ├──────────────────────
    0  2  4  6  8  (seconds)
```

**Leaky vs Token Bucket Comparison:**

```
Scenario: 10 requests arrive, then 10 more after 1 second
Limit: 5 requests/second capacity

Token Bucket (capacity=10, refill=5/sec):
├─ Time 0: 10 requests
│  └─ 10 tokens available → all allowed immediately ✓
├─ Time 1: 10 requests  
│  └─ 5 tokens refilled → 5 allowed, 5 rejected
└─ Result: Allows bursts (all 10 at once)

Leaky Bucket (capacity=10, leak=5/sec):
├─ Time 0: 10 requests
│  └─ All queued (within capacity)
│  └─ Processing at 5/sec → takes 2 seconds to finish
├─ Time 1: 10 requests
│  └─ Queue has 5 remaining from previous
│  └─ Can queue 5 more (5+5=10 capacity reached)
│  └─ Remaining 5 rejected ✗
└─ Result: Smooths traffic (constant outflow rate)
```

**Use Cases:**

```
Leaky Bucket is ideal for:

1. Traffic Smoothing:
   └─ Protect downstream services from bursts
   └─ Example: Message queue processing

2. Network Traffic Shaping:
   └─ Ensure constant bandwidth usage
   └─ Example: Video streaming

3. Background Job Processing:
   └─ Process jobs at predictable rate
   └─ Example: Email sending service

Real Example - Background Worker:
├─ Jobs arrive in bursts (1000 at once)
├─ Worker can handle 10 jobs/second
├─ Leaky bucket queues jobs
└─ Worker processes at steady 10/sec rate
```

**Pros:**

```
✓ Smooths out traffic bursts
✓ Constant processing rate (predictable)
✓ Protects downstream services
✓ Simple concept (queue + drain)
```

**Cons:**

```
✗ No bursts allowed (even if capacity available)
✗ Queue can introduce latency
✗ More complex than token bucket
✗ Older requests can get stale in queue
```

**When to Use:**

```
✓ Need constant output rate
✓ Protecting downstream services from spikes
✓ Background job processing
✗ Don't use when bursts are acceptable
✗ Don't use for interactive APIs (latency)
```

-----

### 4.6 Algorithm Comparison Summary

|Algorithm                 |Memory     |Accuracy   |Bursts|Smoothing|Complexity|Use Case                |
|--------------------------|-----------|-----------|------|---------|----------|------------------------|
|**Fixed Window**          |O(1)       |Low        |No    |No       |Simple    |Internal services       |
|**Sliding Window Log**    |O(limit)   |High       |No    |Yes      |Medium    |Security-critical       |
|**Sliding Window Counter**|O(1)       |Medium-High|No    |Yes      |Medium    |Production (most common)|
|**Token Bucket**          |O(1)       |Medium     |Yes   |Yes      |Medium    |API rate limiting       |
|**Leaky Bucket**          |O(capacity)|High       |No    |Yes      |Medium    |Traffic shaping         |

**Decision Tree:**

```
Do you need to allow bursts?
├─ YES → Token Bucket
│   └─ Allows controlled bursts while limiting sustained rate
│
└─ NO → Need smooth traffic?
    ├─ YES → Need constant output?
    │   ├─ YES → Leaky Bucket (queue-based processing)
    │   └─ NO → Sliding Window Counter (best balance)
    │
    └─ NO → Need perfect accuracy?
        ├─ YES → Sliding Window Log (memory-intensive)
        └─ NO → Fixed Window (simplest)
```

-----

## 5. Distributed Rate Limiting

### 5.1 The Distributed Challenge

**Problem:** Rate limiting across multiple servers

```
Single Server (Easy):
┌──────────────┐
│ Web Server   │
│ ┌──────────┐ │
│ │In-Memory │ │
│ │ Counter  │ │
│ └──────────┘ │
└──────────────┘

Works perfectly!
```

```
Multiple Servers (Hard):
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Server 1     │  │ Server 2     │  │ Server 3     │
│ Counter: 50  │  │ Counter: 40  │  │ Counter: 30  │
└──────────────┘  └──────────────┘  └──────────────┘

Problem:
├─ User limit: 100 requests/minute
├─ User hits Server 1: 50 requests
├─ User hits Server 2: 40 requests  
├─ User hits Server 3: 30 requests
└─ Total: 120 requests (exceeds limit by 20!) ✗
```

**Why This Happens:**

```
Load Balancer distributes requests:
User makes 120 requests
    ↓
Load Balancer (Round Robin)
    ↓
    ├─────────┬─────────┬─────────┐
    ↓         ↓         ↓         ↓
Server 1  Server 2  Server 3  (repeat)
Count=40  Count=40  Count=40

Each server thinks: "40 < 100, so allow" ✓
But total across all servers: 120 > 100 ✗
```

### 5.2 Solution: Centralized Rate Limit Store

**Architecture:**

```
┌────────────┬────────────┬────────────┐
│ Server 1   │ Server 2   │ Server 3   │
└─────┬──────┴─────┬──────┴─────┬──────┘
      │            │            │
      └────────────┼────────────┘
                   │
                   ▼
        ┌──────────────────┐
        │  Redis Cluster   │
        │  (Shared State)  │
        │                  │
        │ user:123 → 75    │
        │ user:456 → 12    │
        └──────────────────┘

All servers share the same counter!
```

**Redis-based Implementation:**

```python
import redis
import time

class DistributedRateLimiter:
    def __init__(self, redis_client, max_requests, window_seconds):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window = window_seconds
    
    def allow_request_fixed_window(self, user_id):
        """Fixed window using Redis"""
        current_window = int(time.time() / self.window)
        key = f"rate_limit:{user_id}:{current_window}"
        
        # Increment counter
        count = self.redis.incr(key)
        
        # Set expiration on first request (TTL = window size)
        if count == 1:
            self.redis.expire(key, self.window)
        
        return count <= self.max_requests
    
    def allow_request_sliding_window(self, user_id):
        """Sliding window log using Redis sorted set"""
        now = time.time()
        key = f"rate_limit:sliding:{user_id}"
        
        # Remove old entries (outside window)
        cutoff = now - self.window
        self.redis.zremrangebyscore(key, 0, cutoff)
        
        # Count current requests
        count = self.redis.zcard(key)
        
        if count < self.max_requests:
            # Add current request with timestamp as score
            self.redis.zadd(key, {str(now): now})
            # Set expiration
            self.redis.expire(key, self.window)
            return True
        
        return False
    
    def allow_request_token_bucket(self, user_id):
        """Token bucket using Redis hash"""
        key = f"rate_limit:token:{user_id}"
        now = time.time()
        
        # Get current bucket state
        pipe = self.redis.pipeline()
        pipe.hget(key, 'tokens')
        pipe.hget(key, 'last_refill')
        tokens, last_refill = pipe.execute()
        
        # Initialize if doesn't exist
        if tokens is None:
            tokens = self.max_requests
            last_refill = now
        else:
            tokens = float(tokens)
            last_refill = float(last_refill)
        
        # Calculate tokens to add
        refill_rate = self.max_requests / self.window
        time_elapsed = now - last_refill
        tokens_to_add = time_elapsed * refill_rate
        tokens = min(self.max_requests, tokens + tokens_to_add)
        
        # Check if request allowed
        if tokens >= 1:
            tokens -= 1
            
            # Update Redis
            pipe = self.redis.pipeline()
            pipe.hset(key, 'tokens', tokens)
            pipe.hset(key, 'last_refill', now)
            pipe.expire(key, self.window * 2)  # Prevent memory leak
            pipe.execute()
            
            return True
        
        return False

# Usage
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
limiter = DistributedRateLimiter(
    redis_client=redis_client,
    max_requests=100,
    window_seconds=60
)

# Check if request allowed
user_id = "user_123"
if limiter.allow_request_fixed_window(user_id):
    print("Request allowed")
else:
    print("Rate limit exceeded")
```

### 5.3 Race Conditions in Distributed Systems

**The Problem:**

```
Redis Operation: INCR (not atomic with check)

Server 1:                    Server 2:
1. GET count = 99            
                             2. GET count = 99
3. Check: 99 < 100 ✓         
                             4. Check: 99 < 100 ✓
5. INCR → count = 100        
                             6. INCR → count = 101

Result: 101 requests (exceeded limit!)
```

**Solution: Lua Scripts (Atomic Operations)**

```lua
-- Redis Lua script for atomic rate limiting
-- Script stored in Redis, executed atomically

local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current_time = tonumber(ARGV[3])

-- Get current count
local current = redis.call('GET', key)

if current == false then
    -- First request - initialize
    redis.call('SET', key, 1, 'EX', window)
    return 1  -- Allowed
elseif tonumber(current) < limit then
    -- Increment count
    redis.call('INCR', key)
    return 1  -- Allowed
else
    -- Limit exceeded
    return 0  -- Rejected
end
```

**Using Lua Script from Python:**

```python
class AtomicDistributedRateLimiter:
    def __init__(self, redis_client, max_requests, window_seconds):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window = window_seconds
        
        # Load Lua script
        self.script = self.redis.register_script("""
            local key = KEYS[1]
            local limit = tonumber(ARGV[1])
            local window = tonumber(ARGV[2])
            
            local current = redis.call('GET', key)
            
            if current == false then
                redis.call('SET', key, 1, 'EX', window)
                return 1
            elseif tonumber(current) < limit then
                redis.call('INCR', key)
                return 1
            else
                return 0
            end
        """)
    
    def allow_request(self, user_id):
        current_window = int(time.time() / self.window)
        key = f"rate_limit:{user_id}:{current_window}"
        
        # Execute Lua script atomically
        result = self.script(
            keys=[key],
            args=[self.max_requests, self.window]
        )
        
        return result == 1
```

**Why Lua Scripts Solve Race Conditions:**

```
Without Lua (Multiple Commands):
Server 1: GET → 99
Server 2: GET → 99
Server 1: INCR → 100
Server 2: INCR → 101 ✗

With Lua (Single Atomic Operation):
Server 1: Execute Lua script → Check + Increment → 100
Server 2: Execute Lua script → Check + Increment → 101 (rejected) ✓

Lua script runs atomically in Redis
└─ No other commands can interleave
└─ Guarantees correctness!
```

### 5.4 Synchronization Strategies

**Strategy 1: Strict Synchronization (Redis)**

```
Every request checks Redis:
├─ Pro: Accurate across all servers
├─ Con: Network latency on every request
└─ Latency: +1-2ms per request

Use when: Accuracy is critical
```

**Strategy 2: Eventual Consistency (Periodic Sync)**

```
Each server maintains local counter:
├─ Check local counter (fast)
├─ Periodically sync to Redis (every 1 second)
├─ Pro: Very fast local checks
├─ Con: Temporary inaccuracy
└─ Inaccuracy: Up to 2x limit for 1 second

Implementation:
┌──────────────┐
│ Server 1     │
│ Local: 45    │ ─┐
│              │  │
└──────────────┘  │
                  ├─> Every 1 sec
┌──────────────┐  │   sync to Redis
│ Server 2     │  │
│ Local: 38    │ ─┤
│              │  │
└──────────────┘  │
                  │
┌──────────────┐  │
│ Server 3     │  │
│ Local: 42    │ ─┘
└──────────────┘
       │
       ▼
┌──────────────┐
│    Redis     │
│ Total: 125   │
└──────────────┘
```

**Strategy 3: Hybrid Approach**

```python
class HybridRateLimiter:
    def __init__(self, redis_client, max_requests, window):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window = window
        self.local_cache = {}
        self.sync_interval = 1  # seconds
        self.last_sync = time.time()
    
    def allow_request(self, user_id):
        now = time.time()
        
        # Sync with Redis periodically
        if now - self.last_sync > self.sync_interval:
            self._sync_to_redis()
            self.last_sync = now
        
        # Check local cache first (fast path)
        if user_id in self.local_cache:
            local_count = self.local_cache[user_id]
            
            # If obviously over limit locally, reject immediately
            if local_count >= self.max_requests:
                return False
            
            # If close to limit, check Redis (accurate)
            if local_count >= self.max_requests * 0.8:
                return self._check_redis(user_id)
        
        # Normal path: increment local counter
        self.local_cache[user_id] = self.local_cache.get(user_id, 0) + 1
        return True
    
    def _check_redis(self, user_id):
        """Accurate check against Redis"""
        # Implementation similar to previous examples
        pass
    
    def _sync_to_redis(self):
        """Sync local counts to Redis"""
        for user_id, count in self.local_cache.items():
            key = f"rate_limit:{user_id}"
            self.redis.incrby(key, count)
        
        # Clear local cache after sync
        self.local_cache.clear()
```

-----

## 6. System Design & Architecture

### 6.1 Where to Place the Rate Limiter?

**Option 1: Application Server (Embedded)**

```
Request → API Gateway → [Rate Limiter] Application Server → Database
                              ↑
                        Inside app code

Pros:
✓ Full context available (user info, request details)
✓ Can customize per endpoint
✓ No extra network hop

Cons:
✗ Rate limiting logic in every service
✗ Code duplication
✗ Hard to update globally
```

**Option 2: API Gateway / Load Balancer (Middleware)**

```
Request → [Rate Limiter] API Gateway → Application Server → Database
               ↑
          Before app code

Pros:
✓ Centralized logic
✓ Protects all backend services
✓ Easy to update rules
✓ Can reject before reaching app

Cons:
✗ Limited context (no user details yet)
✗ Single point of failure
```

**Option 3: Separate Rate Limiting Service**

```
                  ┌─────────────────┐
Request → Gateway │ Rate Limiter    │ → Application Server
               ↓  │  (Side call)    │
               └──│ Redis Cluster   │
                  └─────────────────┘

Pros:
✓ Highly scalable
✓ Reusable across services
✓ Independent deployment

Cons:
✗ Extra network latency
✗ More complex architecture
```

**Best Practice: Layered Approach**

```
┌─────────────────────────────────────────────┐
│ Layer 1: API Gateway                        │
│ - Coarse rate limiting (1000 req/sec)      │
│ - Protects from DDoS                        │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│ Layer 2: Application Service               │
│ - Fine-grained limits (100 req/min)        │
│ - Per-endpoint limits                       │
│ - Per-user tier limits                      │
└─────────────────────────────────────────────┘
```

### 6.2 Complete Architecture Diagram

```
                      ┌───────────┐
                      │  Client   │
                      └─────┬─────┘
                            │
                            ▼
                   ┌────────────────┐
                   │  CDN / DDoS    │
                   │  Protection    │
                   └────────┬───────┘
                            │
                            ▼
┌───────────────────────────────────────────────────┐
│              Load Balancer / API Gateway          │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │  Rate Limiter Layer 1                       │ │
│  │  - Global limits (10K QPS)                  │ │
│  │  - IP-based throttling                      │ │
│  └─────────────────────────────────────────────┘ │
└───────────────────┬───────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │ App    │  │ App    │  │ App    │
   │Server 1│  │Server 2│  │Server 3│
   │        │  │        │  │        │
   │┌──────┐│  │┌──────┐│  │┌──────┐│
   ││Rate  ││  ││Rate  ││  ││Rate  ││
   ││Limit ││  ││Limit ││  ││Limit ││
   ││Layer2││  ││Layer2││  ││Layer2││
   │└───┬──┘│  │└───┬──┘│  │└───┬──┘│
   └────┼───┘  └────┼───┘  └────┼───┘
        │           │           │
        └───────────┼───────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   Redis Cluster       │
        │  (Rate Limit State)   │
        │                       │
        │  Node 1   Node 2      │
        │  Master   Replica     │
        └───────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │  Application Database │
        └───────────────────────┘
```

### 6.3 Rate Limiting Dimensions

**What to Rate Limit On?**

```
1. Per User ID:
   └─ Limit: 100 requests/minute per authenticated user
   └─ Use: Prevent individual user abuse

2. Per IP Address:
   └─ Limit: 1000 requests/hour per IP
   └─ Use: DDoS protection, anonymous users

3. Per API Key:
   └─ Limit: 10,000 requests/day per API key
   └─ Use: Third-party API integrations

4. Per Endpoint:
   └─ Expensive endpoint: 10 req/min
   └─ Cheap endpoint: 1000 req/min
   └─ Use: Protect resource-intensive operations

5. Global:
   └─ Limit: 100K QPS total across all users
   └─ Use: Protect server capacity

6. Combination (Multi-dimensional):
   └─ User X on Endpoint Y: 10 req/min
   └─ Same user on other endpoints: 100 req/min
```

**Implementation Example:**

```python
class MultiDimensionalRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        
        # Define limits for different dimensions
        self.limits = {
            'user': {'max': 100, 'window': 60},      # 100/min per user
            'ip': {'max': 1000, 'window': 3600},     # 1000/hour per IP
            'api_key': {'max': 10000, 'window': 86400}, # 10K/day per key
            'endpoint:/expensive': {'max': 10, 'window': 60},
            'endpoint:/cheap': {'max': 1000, 'window': 60},
        }
    
    def allow_request(self, request_context):
        """
        request_context = {
            'user_id': 'user_123',
            'ip': '192.168.1.1',
            'api_key': 'key_abc',
            'endpoint': '/expensive'
        }
        """
        # Check all applicable dimensions
        checks = []
        
        # User-based limit
        if 'user_id' in request_context:
            checks.append(('user', request_context['user_id']))
        
        # IP-based limit
        if 'ip' in request_context:
            checks.append(('ip', request_context['ip']))
        
        # API key limit
        if 'api_key' in request_context:
            checks.append(('api_key', request_context['api_key']))
        
        # Endpoint limit
        if 'endpoint' in request_context:
            endpoint_key = f"endpoint:{request_context['endpoint']}"
            if endpoint_key in self.limits:
                checks.append((endpoint_key, request_context['endpoint']))
        
        # Check each dimension
        for dimension, identifier in checks:
            if not self._check_limit(dimension, identifier):
                return False, f"Rate limit exceeded for {dimension}"
        
        return True, "Allowed"
    
    def _check_limit(self, dimension, identifier):
        """Check single dimension limit"""
        if dimension not in self.limits:
            return True
        
        config = self.limits[dimension]
        key = f"rate_limit:{dimension}:{identifier}"
        
        # Use fixed window for simplicity
        current_window = int(time.time() / config['window'])
        redis_key = f"{key}:{current_window}"
        
        count = self.redis.incr(redis_key)
        if count == 1:
            self.redis.expire(redis_key, config['window'])
        
        return count <= config['max']
```

### 6.4 Response Headers & User Experience

**Standard HTTP Headers:**

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000
Retry-After: 60

{
  "error": "Rate limit exceeded",
  "message": "You have exceeded 100 requests per minute",
  "retry_after_seconds": 60
}
```

**Header Meanings:**

```
X-RateLimit-Limit: 100
└─ Maximum requests allowed in window

X-RateLimit-Remaining: 0
└─ How many requests left in current window

X-RateLimit-Reset: 1640000000
└─ Unix timestamp when limit resets

Retry-After: 60
└─ Seconds until user can retry
```

**Implementation:**

```python
from flask import Flask, jsonify, request
import time

app = Flask(__name__)
limiter = DistributedRateLimiter(...)

@app.before_request
def rate_limit_check():
    user_id = request.headers.get('X-User-ID', request.remote_addr)
    
    # Get current limit status
    allowed, remaining, reset_time = limiter.check_limit(user_id)
    
    # Add headers to response
    @app.after_request
    def add_rate_limit_headers(response):
        response.headers['X-RateLimit-Limit'] = '100'
        response.headers['X-RateLimit-Remaining'] = str(remaining)
        response.headers['X-RateLimit-Reset'] = str(reset_time)
        return response
    
    if not allowed:
        retry_after = reset_time - int(time.time())
        return jsonify({
            'error': 'Rate limit exceeded',
            'retry_after_seconds': retry_after
        }), 429, {
            'Retry-After': str(retry_after),
            'X-RateLimit-Limit': '100',
            'X-RateLimit-Remaining': '0',
            'X-RateLimit-Reset': str(reset_time)
        }

@app.route('/api/data')
def get_data():
    return jsonify({'data': 'your data here'})
```

-----

## 7. Advanced Topics

### 7.1 Tiered Rate Limiting

**Different Limits for Different User Tiers:**

```
User Tiers:
├─ Free: 100 requests/hour
├─ Basic: 1,000 requests/hour
├─ Pro: 10,000 requests/hour
└─ Enterprise: 100,000 requests/hour

Implementation:
```

```python
class TieredRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.tier_limits = {
            'free': {'max': 100, 'window': 3600},
            'basic': {'max': 1000, 'window': 3600},
            'pro': {'max': 10000, 'window': 3600},
            'enterprise': {'max': 100000, 'window': 3600},
        }
    
    def get_user_tier(self, user_id):
        """Fetch user's subscription tier from database"""
        # In production: query database or cache
        # For example: return db.query("SELECT tier FROM users WHERE id = ?", user_id)
        return 'free'  # Placeholder
    
    def allow_request(self, user_id):
        tier = self.get_user_tier(user_id)
        limit_config = self.tier_limits.get(tier, self.tier_limits['free'])
        
        # Apply rate limit based on tier
        key = f"rate_limit:{tier}:{user_id}"
        current_window = int(time.time() / limit_config['window'])
        redis_key = f"{key}:{current_window}"
        
        count = self.redis.incr(redis_key)
        if count == 1:
            self.redis.expire(redis_key, limit_config['window'])
        
        return count <= limit_config['max']
```

### 7.2 Adaptive Rate Limiting

**Dynamically Adjust Limits Based on System Load:**

```python
class AdaptiveRateLimiter:
    def __init__(self, redis_client, base_limit=1000):
        self.redis = redis_client
        self.base_limit = base_limit
    
    def get_system_load(self):
        """Monitor system metrics"""
        # In production: query monitoring system (Prometheus, CloudWatch)
        cpu_usage = 0.75  # 75% CPU
        memory_usage = 0.60  # 60% memory
        queue_depth = 100  # 100 pending requests
        
        return {
            'cpu': cpu_usage,
            'memory': memory_usage,
            'queue': queue_depth
        }
    
    def calculate_dynamic_limit(self):
        """Adjust limit based on system health"""
        metrics = self.get_system_load()
        
        # If system overloaded, reduce limits
        if metrics['cpu'] > 0.8 or metrics['memory'] > 0.8:
            multiplier = 0.5  # Reduce to 50%
        elif metrics['cpu'] > 0.6 or metrics['memory'] > 0.6:
            multiplier = 0.75  # Reduce to 75%
        else:
            multiplier = 1.0  # Normal limits
        
        return int(self.base_limit * multiplier)
    
    def allow_request(self, user_id):
        current_limit = self.calculate_dynamic_limit()
        
        # Use current limit for rate limiting
        # ... (similar to previous implementations)
```

**Visualization:**

```
System Load vs Rate Limit:

Limit
1000 │          ████████████
     │      ████
 750 │  ████
     │██
 500 │
     │
   0 ├────────────────────────
     0   20   40   60   80  100 (% CPU)

When CPU usage increases:
└─ Automatically reduce rate limits
└─ Protects system from overload
└─ Gradually increase when load decreases
```

### 7.3 Rate Limiting in Microservices

**Challenge: Each Service Needs Rate Limiting**

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Service  │───▶│ Service  │───▶│ Service  │
│    A     │    │    B     │    │    C     │
│(Gateway) │    │ (Core)   │    │  (DB)    │
└──────────┘    └──────────┘    └──────────┘
     │               │               │
     └───────────────┼───────────────┘
                     │
              ┌──────▼──────┐
              │    Redis    │
              │ (Shared)    │
              └─────────────┘

Problem:
├─ Service A → Service B: Need rate limit
├─ Service B → Service C: Need rate limit
└─ User → Service A: Need rate limit

Solution: Hierarchical Rate Limiting
```

**Implementation:**

```python
class MicroserviceRateLimiter:
    def __init__(self, redis_client, service_name):
        self.redis = redis_client
        self.service_name = service_name
        
        # Define limits per service pair
        self.service_limits = {
            'gateway->auth': {'max': 1000, 'window': 60},
            'gateway->core': {'max': 500, 'window': 60},
            'core->database': {'max': 100, 'window': 60},
        }
    
    def allow_request(self, from_service, to_service, identifier):
        """
        Check rate limit for inter-service call
        
        from_service: Calling service
        to_service: Target service
        identifier: Request identifier (user_id, request_id, etc.)
        """
        service_pair = f"{from_service}->{to_service}"
        
        if service_pair not in self.service_limits:
            return True  # No limit defined
        
        limit_config = self.service_limits[service_pair]
        key = f"rate_limit:service:{service_pair}:{identifier}"
        
        # Apply rate limit
        current_window = int(time.time() / limit_config['window'])
        redis_key = f"{key}:{current_window}"
        
        count = self.redis.incr(redis_key)
        if count == 1:
            self.redis.expire(redis_key, limit_config['window'])
        
        return count <= limit_config['max']
```

### 7.4 Rate Limiting for DDoS Protection

**Multi-Layer Defense:**

```
Layer 1: Infrastructure (Cloudflare, AWS Shield)
├─ Filter malicious traffic
├─ Block known bad IPs
└─ Rate limit: 100K QPS per IP

Layer 2: API Gateway
├─ Geographic rate limiting
├─ Rate limit: 10K QPS per region
└─ Detect anomalous patterns

Layer 3: Application
├─ User-based rate limiting
├─ Endpoint-specific limits
└─ Behavior analysis

Layer 4: Database
├─ Query rate limiting
├─ Connection pooling
└─ Read replica distribution
```

**Anomaly Detection:**

```python
class AnomalyDetectionRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.baseline_window = 3600  # 1 hour
    
    def detect_anomaly(self, user_id):
        """Detect if user behavior is anomalous"""
        # Get historical average
        historical_key = f"baseline:{user_id}"
        baseline = self.redis.get(historical_key)
        
        if baseline is None:
            baseline = 10  # Default
        else:
            baseline = float(baseline)
        
        # Get current rate
        current_key = f"current:{user_id}"
        current_rate = self.redis.get(current_key) or 0
        current_rate = float(current_rate)
        
        # If current rate >> baseline, it's anomalous
        if current_rate > baseline * 10:  # 10x normal rate
            return True, "Anomalous traffic detected"
        
        return False, "Normal"
    
    def update_baseline(self, user_id, current_rate):
        """Update user's baseline (rolling average)"""
        key = f"baseline:{user_id}"
        
        # Exponential moving average
        alpha = 0.1
        old_baseline = self.redis.get(key) or current_rate
        old_baseline = float(old_baseline)
        
        new_baseline = alpha * current_rate + (1 - alpha) * old_baseline
        
        self.redis.set(key, new_baseline, ex=86400)  # 24-hour TTL
```

-----

## 8. Real-World Case Studies

### 8.1 GitHub API Rate Limiting

**Implementation:**

```
Unauthenticated: 60 requests/hour per IP
Authenticated: 5,000 requests/hour per user
Enterprise: 15,000 requests/hour

Algorithm: Token Bucket (allows bursts)

Headers:
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640000000
X-RateLimit-Used: 1
X-RateLimit-Resource: core
```

**Why Token Bucket:**

```
GitHub allows developers to:
├─ Make occasional bursts (clone large repos)
├─ Sustained usage limited to 5K/hour
└─ Token bucket perfect for this pattern
```

### 8.2 Stripe API Rate Limiting

**Implementation:**

```
Rate limits:
├─ Standard: 100 requests/second
├─ Bursts allowed up to 1,000 requests
├─ Uses Token Bucket algorithm

Special handling:
├─ Higher limits for trusted partners
├─ Gradual backoff for violations
└─ Whitelist for critical integrations
```

**Response:**

```http
HTTP/1.1 429 Too Many Requests
Stripe-RateLimit-Limit: 100
Stripe-RateLimit-Remaining: 0
Stripe-RateLimit-Reset: 1640000000

{
  "error": {
    "type": "rate_limit_error",
    "message": "Too many requests. Please slow down."
  }
}
```

### 8.3 Twitter API Rate Limiting

**Implementation:**

```
Different endpoints, different limits:
├─ GET tweets/search: 180 requests / 15 min
├─ POST tweets/create: 300 requests / 3 hours
├─ GET users/lookup: 900 requests / 15 min

Window: 15 minutes (sliding window)
Algorithm: Sliding Window Counter

Per-endpoint tracking:
└─ Each endpoint has separate counter
```

**Why Different Limits:**

```
Read operations (GET): Higher limits
├─ Less resource-intensive
├─ Cached responses
└─ Example: 900 req / 15 min

Write operations (POST): Lower limits
├─ More expensive (database writes)
├─ Spam prevention
└─ Example: 300 req / 3 hours

This protects backend while allowing flexibility
```

-----

## 9. Interview Framework

### 9.1 How to Approach “Design a Rate Limiter”

**Step 1: Requirements Clarification (3-5 min)**

```
Questions to Ask:

Functional:
├─ What type of rate limiter? (User, IP, API key?)
├─ What's the rate? (100/min, 1000/hour, etc.)
├─ Distributed or single server?
├─ Hard block or throttle? (reject vs delay)

Non-Functional:
├─ Scale: How many users? Requests?
├─ Latency: What overhead is acceptable?
├─ Accuracy: Strict or approximate?
└─ Fault tolerance: Fail open or closed?

Example Answer:
"Let me clarify:
- We're rate limiting API requests per user
- Limit: 100 requests per minute
- Distributed system (multiple servers)
- Hard block (return 429 error)
- Scale: 10M users, 100K QPS
- Latency: < 1ms overhead
- Accuracy: 99%+ (some edge case tolerance OK)
Is this correct?"
```

**Step 2: Choose Algorithm (2-3 min)**

```
Decision Framework:

Need bursts?
├─ YES → Token Bucket
└─ NO → Need perfect accuracy?
    ├─ YES → Sliding Window Log
    └─ NO → Sliding Window Counter (recommended!)

Explain your choice:
"I'll use Sliding Window Counter because:
✓ Memory efficient: O(1) per user
✓ Good accuracy (no 2x boundary issue)
✓ Fast: O(1) operations
✓ Production-proven (used by Twitter, Stripe)

Trade-off: Slight estimation vs perfect accuracy
└─ But 99%+ accuracy is acceptable for this use case"
```

**Step 3: High-Level Design (5 min)**

```
Draw Architecture:

┌─────────────────────────────────────┐
│ Client                              │
└───────────────┬─────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│ API Gateway / Load Balancer         │
│ - Rate limit check (Layer 1)       │
│ - IP-based: 1000 req/sec           │
└───────────────┬─────────────────────┘
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
┌─────────────┐ ┌─────────────┐
│ App Server 1│ │ App Server 2│
└──────┬──────┘ └──────┬──────┘
       │               │
       └───────┬───────┘
               │
               ▼
┌──────────────────────────────────┐
│ Redis Cluster (Rate Limit State) │
│ - Sliding window counters        │
│ - user:123 → {prev: 80, curr: 30}│
└──────────────────────────────────┘

Request Flow:
1. Request → Load Balancer
2. Check Redis for user's rate limit
3. If allowed → forward to app server
4. If rejected → return 429
```

**Step 4: Deep Dive (15-20 min)**

**A) Algorithm Implementation:**

```
Explain Sliding Window Counter:

"At time 01:15 (15 seconds into window 2):

Window 1 [00:00-01:00]: 80 requests
Window 2 [01:00-02:00]: 30 requests (so far)

Calculate:
├─ Time in current window: 15 seconds (25%)
├─ Weight previous window: 1 - 0.25 = 0.75
├─ Estimate: (80 × 0.75) + 30 = 60 + 30 = 90
└─ 90 < 100 → Allow ✓

Redis data structure:
{
  'user:123:window:1': 80,
  'user:123:window:2': 30
}

On new request:
1. Incr current window counter
2. Check weighted sum
3. Return allow/deny
"
```

**B) Distributed System Handling:**

```
"Challenge: Multiple servers accessing Redis

Race condition:
Server 1: READ count=99 → INCR → count=100 ✓
Server 2: READ count=99 → INCR → count=101 ✗

Solution: Lua Script (atomic)
─────────────────────────
local count = redis.call('GET', key)
if count < limit then
  redis.call('INCR', key)
  return 1  -- allowed
else
  return 0  -- rejected
end
─────────────────────────

This runs atomically in Redis
└─ Prevents race conditions"
```

**C) Response Headers:**

```
"Return helpful headers:

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000
Retry-After: 45

Benefits:
├─ User knows limit (100)
├─ User knows when to retry (45 sec)
└─ Better developer experience"
```

**Step 5: Optimizations (5 min)**

```
Optimizations to Discuss:

1. Hot User Problem:
   "Celebrity users hit same Redis key
   
   Solution:
   ├─ Local cache (L1) in app servers
   ├─ Cache limit state for 1 second
   └─ Reduces Redis load by 10-100x"

2. Redis Failure:
   "What if Redis goes down?
   
   Options:
   ├─ Fail Open: Allow all requests (availability)
   ├─ Fail Closed: Reject all (security)
   └─ Hybrid: Use local fallback counters (temporary)"

3. Memory Optimization:
   "Clean up old entries:
   ├─ Set TTL on Redis keys (auto-expire)
   ├─ Cleanup job removes inactive users
   └─ Prevents memory leak"
```

**Step 6: Monitoring (2 min)**

```
"Key Metrics:

1. Rejection Rate:
   └─ % of requests rejected
   └─ Alert if suddenly spikes

2. Top Rate-Limited Users:
   └─ Identify potential abusers
   └─ Or legitimate users needing upgrade

3. Rate Limiter Latency:
   └─ p99 latency < 1ms
   └─ Redis response time

4. False Positive Rate:
   └─ Legitimate requests rejected
   └─ Should be < 0.1%"
```

### 9.2 Common Follow-up Questions

**Q1: “How would you handle different rate limits for different user tiers (free vs paid)?”**

```
Answer:

"Store user tier in cache/database:

user_id → tier (free/pro/enterprise)

Rate limit logic:
├─ Fetch user tier
├─ Apply tier-specific limit
│  ├─ Free: 100 req/min
│  ├─ Pro: 1000 req/min
│  └─ Enterprise: 10000 req/min
└─ Use same algorithm (sliding window)

Redis key includes tier:
rate_limit:{tier}:{user_id}:{window}

This allows easy tier upgrades:
└─ Change tier → new limits apply immediately"
```

**Q2: “User complains they’re being rate limited unfairly. How do you debug?”**

```
Answer:

"Debugging Steps:

1. Check Logs:
   └─ Look up user's request history
   └─ Verify actual request count

2. Check Redis State:
   └─ Redis CLI: GET rate_limit:user:123:*
   └─ See current counters

3. Check Clock Skew:
   └─ Are servers in sync?
   └─ Time drift can cause issues

4. Check for Multiple Accounts:
   └─ Same IP, different user IDs?
   └─ Shared API key?

5. Verify Rate Limit Rules:
   └─ Is limit configured correctly?
   └─ Did rules change recently?

If legitimate:
├─ Temporarily increase limit
├─ Whitelist user
└─ Investigate algorithm accuracy"
```

**Q3: “A new feature launches and traffic spikes 10x. What happens to rate limiting?”**

```
Answer:

"Immediate Impact:
├─ More users hit rate limits
├─ Redis load increases
└─ Potential system overload

Short-term Response:
├─ Monitor rejection rate
├─ Consider temporary limit increase
├─ Add more Redis nodes if needed
└─ Enable request prioritization

Long-term Solution:
├─ Adaptive rate limiting
│  └─ Adjust limits based on system load
├─ Graceful degradation
│  └─ Prioritize critical endpoints
└─ Capacity planning
   └─ Scale infrastructure proactively

Trade-off Discussion:
'We could:
A) Keep strict limits (protect system)
B) Temporarily relax (better UX)
C) Tier-based (free users limited, paid OK)

I'd choose C because:
✓ Protects system
✓ Good UX for paying customers
✓ Incentivizes upgrades'"
```

### 9.3 Red Flags to Avoid

**❌ Don’t Say:**

```
1. "Just use fixed window, it's simple"
   ↳ Shows lack of depth (boundary problem)

2. "Store all timestamps in memory"
   ↳ Doesn't scale (memory explosion)

3. "Rate limit in database"
   ↳ Too slow (defeats purpose)

4. "Don't need distributed state"
   ↳ Ignores multi-server reality

5. "Block attackers permanently"
   ↳ Too harsh (could be legitimate spike)
```

**✅ Do Say:**

```
1. "I'll use sliding window counter because:
   - Good accuracy without memory overhead
   - Production-proven at scale
   - Trade-off: slight estimation vs perfect accuracy"

2. "For 10M users × 100 requests:
   - Sliding window log: 1M × 100 × 8 bytes = 800MB
   - Sliding window counter: 1M × 2 counters × 8 bytes = 16MB
   - Clear winner: counter approach"

3. "Rate limiter needs < 1ms latency:
   - In-memory cache (Redis) required
   - Database too slow (10-50ms)
   - Network round-trip to Redis: ~1ms acceptable"

4. "Multiple servers require shared state:
   - Redis cluster for coordination
   - Lua scripts for atomic operations
   - Prevents race conditions"

5. "For DDoS protection:
   - Layer 1: Infrastructure (Cloudflare)
   - Layer 2: IP rate limiting (gateway)
   - Layer 3: User rate limiting (app)
   - Layer 4: Graceful degradation (fallback)"
```

-----

## Summary Checklist

Before your interview, ensure you can explain:

**Core Concepts:**

- [ ] Why rate limiting is needed (protection, fairness, cost)
- [ ] The boundary problem in fixed window
- [ ] Why distributed systems need shared state

**Algorithms:**

- [ ] Fixed Window (simple but boundary problem)
- [ ] Sliding Window Log (accurate but memory-heavy)
- [ ] Sliding Window Counter (best balance, production choice)
- [ ] Token Bucket (allows bursts)
- [ ] Leaky Bucket (smooths traffic)

**Implementation:**

- [ ] Redis-based distributed rate limiting
- [ ] Lua scripts for atomic operations
- [ ] Race condition prevention
- [ ] Response headers (X-RateLimit-*)

**System Design:**

- [ ] Where to place rate limiter (gateway vs app)
- [ ] Multi-dimensional rate limiting (user, IP, endpoint)
- [ ] Tiered limits (free vs paid)
- [ ] Failure modes (fail open vs closed)

**Advanced Topics:**

- [ ] DDoS protection strategies
- [ ] Adaptive rate limiting
- [ ] Microservices rate limiting
- [ ] Monitoring and debugging

**Interview Skills:**

- [ ] Requirements clarification questions
- [ ] Algorithm selection justification
- [ ] Drawing clear architecture diagrams
- [ ] Explaining trade-offs
- [ ] Handling follow-up questions

-----

## Next Steps

**Practice:**

1. Implement sliding window counter in your language
1. Draw the architecture from memory
1. Explain token bucket to someone else
1. Walk through distributed race condition scenario

**Additional Topics to Explore:**

- Circuit Breaker pattern (related concept)
- API Gateway implementations (Kong, Nginx)
- Redis clustering and replication
- Prometheus metrics for rate limiting

**Related System Design Problems:**

- API Gateway design
- DDoS protection system
- Load Balancer design
- Throttling and backpressure

-----

**Good luck with your interviews!** Remember: Rate limiting is about trade-offs between accuracy, performance, and user experience. Show your thinking process, explain your choices, and discuss alternatives.