---
layout: post
title: Understanding AWS RDS Burst Balance
tags:
  - AWS
  - RDS
---

## The Story
Recently the RDS I was managing started to use a lot more burst balance than usual. In order to understand what could be done to address it, as well as how it would affect the performance of the database, I went to study the [AWS documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html) about how burst balance works.

After reading the documentation for the first time, I didn't fully understand how burst balance works, as well as the effects it has on IOPS and database performance. There is a lot to digest. All the numbers and math definitely didn't help.

But now, I think I have a good understand of it. So, I'm going to try to make it a little easier to understand by breaking it down into key points, and skipping over some information that I think isn't that useful. I'll also be throwing in an analogy to make it a lot easier to wrap your head around burst balance, especially for the less technically inclined folks.

## What Burst Balance Applies To
- Burst balance only applies to General Purpose SSD storage (gp2). It does not apply to Magnetic storage, or Provisioned IOPS (io1).

## Baseline Performance
- Every SSD comes with a baseline performance, measured in IOPS (I/O (input/output) per second).
- Each GB of storage provides 3 IOPS of baseline performance.
  - There is a minimum of 100 IOPS. So any SSD between 20-33 GB will have a baseline performance of 100 IOPS.
  - There is a maximum of 16,000 IOPS. So any SSD above 5340 GB will have a baseline performance of 16,000 IOPS.
  - Examples: 500 GB has 1,500 IOPS, 1,000 GB has 3,000 IOPS.
- This baseline performance is guaranteed for your SSD. You will have this level of IOPS even if you have no burst balance remaining.

## Burst
- Bursting is having your SSD go over the baseline performance for a certain period of time.
- The burst duration is dependent on the burst balance, which is related to how much it's gone over the baseline, as well as the size of the SSD.
- SSDs below 1,000 GB are able to burst to a maximum of 3,000 IOPS.
- SSDs that are 1,000 GB and above do not have a maximum IOPS that they can burst to.
  - For this, the documentation says that burst is not relevant for SSDs above 1,000 GB. The phrase "not relevant" can be a little vague. Does it mean there's no burst at all? Or does it mean it can burst to any value? From personal experience, the highest IOPS value I have seen my 1,000 GB SSD burst to is 15,000 IOPS).

## Burst Balance
- When the IOPS usage of your SSD goes above the baseline, burst balance is used.
- When the IOPS usage goes below the baseline, burst balance is regenerated.
- If your burst balance reaches 0%, your SSD will not perform any higher than the baseline IOPS. This can cause severe issues with your application performance if it relies too much on burst.
- I feel that the actual calculations for this aren't that important to know, because I don't think you should be planning too much around using the burst balance due to the dangers of hitting 0%. Burst balance should be treated as a safety net, not something to be used regularly.
- Actual math anyway:
  - All SSDs get an initial credit balance of 5.4 million IOPS.
  - Credits are earned at 3 IOPS per GB per second.
  - So, a 200 GB SSD earns 600 IOPS per second.
  - The burst duration formula is given in the [AWS documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html#CHAP_Storage.IO.Credits):
  ```
                              (Credit balance)
   Burst duration =  --------------------------------------
                     (Burst IOPS) - 3*(Storage size in GiB)
  ```
  - Even with this formula, the burst duration isn't accurate due to other factors:
    - The burst IOPS is usually not constant.
    - If your IOPS usage dips below your baseline, you start gaining credit balance.
    - So, just don't bother with this. Really.

## Analogy
After reading all that, it can be rather difficult to wrap your head around it. How do you understand it intuitively? What's a good way to visualize how this works?

Here's the analogy. Unfortunately you would need a little familiarity with racing games with the nitrous boost feature to really get this.

First, some terms:
- Car = RDS
- Nitrous boost = burst
- Nitrous amount = burst balance
- Speed = IOPS

Each car has a speed limit it can reach. Let's say it's 200km/h. This is this baseline performance of the car.

Nitrous allows the car to boost beyond this limit, maybe to 300km/h for 10 seconds, then the nitrous is depleted and the car cannot go above 200km/h no matter how hard it tries.

It can't go beyond 200km/h anymore, because the nitrous is depleted.

If the car drives below 200km/h, then the nitrous regenerates, and we can use it to make the car can go beyond 200km/h again. The slower the car goes, the faster the nitrous would regenerate.

Boosting with nitrous isn't a fixed thing; 300km/h for 10 seconds was just one way to use it. The car might decide to boost to 400km/h for 5 seconds, or 250km/h for 15 seconds. The only rule is that the higher the speed, the shorter the duration. You could even decide to go to 400km/h for 3 seconds, drop back to 150km/h for a while to regenerate some nitrous, then go up to 400km/h again, and keep repeating this cycle. Maybe this specific cycle would completely deplete the nitrous, or maybe it wouldn't. There's a sweet spot to this.

Let's say we decided to upgrade our car and increase the speed limit to 400km/h (by increasing the disk size). Now, if we want to drive at 400km/h, we don't even have to spend any nitrous. And to regenerate nitrous, now we only have to stay below 400km/h, not 200km/h.

Note that the old car and the upgraded car are both capable of reaching high speeds. They don't have a difference in the max speed they can reach while boosting. Both can reach 1000km/h if they want to.
The only difference is how long their nitrous tank can last. (This paragraph applies only for disk sizes below 1,000 GB. Sizes below 1,000 GB can only burst up to 3,000 IOPS. Effectively a speed limit for their nitrous.)

Hope this helps!