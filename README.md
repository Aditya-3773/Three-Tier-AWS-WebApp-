# Three-Tier Web App on AWS

I built this to actually understand how a "real" AWS architecture fits together — not just spin up an EC2 instance and call it a day, but build the full public/private separation you'd see in a production setup: load balancer out front, app servers in the middle, database locked away in the back.

Everything here was built by hand in the AWS Console, click by click, on purpose — no Terraform, no CloudFormation. I wanted to actually understand what each piece does before automating any of it away.

**Traffic flow:** Internet -> Application Load Balancer (public) -> EC2 app tier / Auto Scaling Group (private) -> RDS (private)

---

## Architecture

The whole design comes down to one rule: **each layer only trusts the layer directly in front of it.**

- The **public subnets** are the only part of the VPC that can talk to the internet directly — that's where the ALB and NAT Gateway live.
- The **private app subnets** hold the EC2 instances. They have no public IP at all, and the only thing allowed to reach them is the load balancer.
- The **private DB subnets** hold RDS, and the only thing allowed to reach *that* is the app tier.
- **S3** sits off to the side for static assets/backups, and the app reaches it through an IAM role instead of the bucket being open to the world.

So even if someone found the app server's private IP somehow, they still couldn't reach the database — it only listens for traffic coming from the app tier's security group, nothing else.

---

## What's actually running

| Service | What it's doing here |
|---|---|
| VPC | The whole isolated network, `10.0.0.0/16`, spread across `us-east-1a` and `us-east-1b` |
| 6 Subnets | 2 public, 2 private for the app, 2 private for the database |
| Internet Gateway | Gives the public subnets a way in/out to the internet |
| NAT Gateway | Lets the private subnets reach *out* to the internet without being reachable from it |
| Route Tables | This is what actually makes a subnet "public" or "private" — it's just which route table it's tied to |
| Security Groups | ALB → App → DB, each one only trusting the one before it |
| EC2 + Auto Scaling Group | The app tier — private subnets, no public IP, self-healing if an instance dies |
| RDS (MySQL, Multi-AZ) | The database tier — private, public access turned off entirely |
| Application Load Balancer | The one public-facing entry point into the whole thing |
| S3 | Static assets/backups, private bucket, only the app's IAM role can touch it |
| SSM Session Manager | How I actually get into the private EC2 instances — no SSH key, no bastion host, no port 22 open anywhere |

---

## The subnet math

I planned this out on paper before touching the console, mostly so I wouldn't second-guess myself halfway through:

| Component | CIDR |
|---|---|
| VPC | `10.0.0.0/16` |
| Public subnet A | `10.0.0.0/24` |
| Public subnet B | `10.0.1.0/24` |
| Private app subnet A | `10.0.10.0/24` |
| Private app subnet B | `10.0.11.0/24` |
| Private DB subnet A | `10.0.20.0/24` |
| Private DB subnet B | `10.0.21.0/24` |

Two AZs each, so a single zone going down doesn't take the whole thing with it.

---

## Security groups, chained

This part is really the heart of the whole project:

- **`alb-sg`** — open to the internet on HTTP/HTTPS, since that's the whole point of a load balancer
- **`app-sg`** — only accepts traffic *from* `alb-sg`, nothing else
- **`db-sg`** — only accepts traffic *from* `app-sg`, nothing else

I didn't just take this on faith — I tried to break it on purpose:

- Tried SSH-ing straight into a private EC2 instance from my laptop → refused, as expected, no public IP and nothing allowing it in
- Tried hitting the RDS endpoint directly from my laptop → refused too, no public access and the security group won't allow it

Those two failures were honestly more satisfying to see than the app actually working — that's the whole security model doing its job.

---

## How I built it

Roughly, in this order:

1. Created the VPC and all 6 subnets across two AZs
2. Set up the Internet Gateway and attached it
3. Set up a NAT Gateway with an Elastic IP in one of the public subnets
4. Built the route tables and attached them to the right subnets
5. Created the three security groups and chained them (ALB → app → DB)
6. Launched EC2 in the private app subnets through an Auto Scaling Group — connected via SSM Session Manager, never SSH
7. Set up RDS in the private DB subnets, public access off
8. Set up the ALB in the public subnets, pointing at the app tier
9. Added an S3 bucket for static assets, kept private, scoped IAM access only
10. Tested the whole thing end to end — and then tried to break in from outside to confirm it couldn't be

---

## Proof it actually works

**The app responds through the load balancer** — hitting the ALB's DNS name in a browser loads the app correctly, no direct access to any backend piece required.

**The app tier can actually talk to the database.** I connected to the app-tier EC2 instance through SSM Session Manager (no SSH key, no exposed port) and ran this straight against RDS:

```sql
USE appdb;
CREATE TABLE test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(100)
);
INSERT INTO test (message) VALUES ('Data inserted from ASG EC2');
SELECT * FROM test;
```

It worked — which confirms an instance sitting in a private subnet, with no public IP of its own, can still reach the database sitting in *its* private subnet, but strictly through the `app-sg → db-sg` chain and nothing else.

**The Auto Scaling Group stayed healthy** the whole time I was testing — instances passed their checks and were spread across both AZs, so losing one wouldn't have taken the app down.

---

## Don't forget to tear it down

NAT Gateway and RDS both bill by the hour, so I didn't leave them running between sessions. Order matters here — AWS won't let you delete something another resource still depends on, so:

1. ALB
2. Auto Scaling Group / EC2 instances
3. RDS
4. NAT Gateway — then go release the Elastic IP separately, it doesn't always let go on its own
5. Subnets
6. The custom route tables (leave the default one, it goes with the VPC)
7. Internet Gateway — detach first, then delete
8. VPC last, once everything inside it is gone

---

## What I actually took away from this

- Public vs. private isn't a setting on a subnet — it's just which route table it's attached to
- Security groups chained by tier are a much better mental model than "allow this IP range"
- SSM Session Manager is genuinely nicer than managing SSH keys and bastion hosts
- An Auto Scaling Group is barely more setup than a single EC2 instance, and you get self-healing for free
- Knowing the *teardown* order matters just as much as knowing the build order — it's really the same dependency graph either way

---

## Where I'd take this next

- Add HTTPS on the ALB with an ACM certificate
- Rebuild this same architecture in Terraform or CloudFormation now that I understand what every piece is actually doing
- Add some basic CloudWatch alarms
- Put CloudFront in front of S3 if I ever need to serve static assets publicly
