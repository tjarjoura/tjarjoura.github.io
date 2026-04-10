---
layout: post
title: "Assumptions Your SaaS Application Makes That Prevent You From Deploying On-Prem"
date: 2026-04-09
---

SaaS is the default delivery method for many software companies, for good reason, but it can totally shut you out from entire market segments.

Organizations in segments like government, healthcare, and finance all have extensive legal regulations which make on-prem a de-facto requirement. They could potentially benefit from your software but need you to package it in a way that works with this on-prem requirement.

In order to achieve this, it's important to avoid certain assumptions that wouldn't hold in an on-premises environment. Any of these assumptions could end up being a future dealbreaker for a lucrative contract.

Here's a few examples that I used to take for granted when I had unlimited access to whatever cloud services I wanted and had unlimited time to fine-tune the singleton SaaS environment to my liking. The purpose is not to be comprehensive, but just to get you thinking.

## Network Connectivity

Your SaaS offering might assume it has free rein to access the internet. Maybe it tries to access a third party service hosted on the internet -- cloud hosted email providers, payment processors, telemetry collectors, etc. Maybe your install/deploy process tries to download dependencies from the internet.

Any of these can result in network connectivity errors, where your application hangs for 30 seconds on startup and then crashes with a cryptic error message.

You should identify every outbound network dependency that your application has. It's probably a lot more than you think. Then as much as possible think about an internal solution that could replace these, or make the dependency optional and think about a way for your application to handle it gracefully.

## Secret Management

Your SaaS offering might assume it can pull secrets from AWS Secrets Manager, a managed Hashicorp Vault instance, or Google Cloud Secrets. On prem these secrets need to be pulled from some secure storage inside the customer's environment.

Audit your code for assumptions about where it pulls secrets from. Add support for general purpose injection methods like environment variables, or bundle your own secret management solution. Make it easy for the customer to provide the secret inputs your application needs without exposing them to unnecessary security risks.

## Certificate Management

Six months after you deploy to the customer's environment, they start getting confusing errors about certificate expiration. Your SaaS application used Let's Encrypt, and certificate rotation was handled automatically, so no one thought about this problem in the customer's environment. 

Additionally, your SaaS offering might serve TLS with a certificate obtained from a public CA, but your on-prem customer might have their own internal CA they want you to use, or maybe you need to come with a solution to package some kind of mini-PKI with your application. 

## Runtime

Your SaaS offering might run on cloud hosted Kubernetes. On-prem, the customer might not have a Kubernetes cluster, or might have their own in house cluster. Or maybe you need to ship a self-hosted Kubernetes distribution like K3s, which may have different constraints than your cloud-hosted Kubernetes.

Ideally you should have integration tests which spin up your application from scratch in the type of environment you expect your customer to have.

## Authentication

Your SaaS offering might have integrations with cloud hosted identity providers like Google, but on-prem customers might want to authenticate users with LDAP or Kerberos (Active Directory). Try to keep your authentication methods de-coupled from the rest of your business logic.

This is potentially a more complex undertaking than the others, but could really move the needle with enterprise buyers.

## DNS Records

Your SaaS offering is available at `my-cool-saas.com`. On-prem it becomes `my-cool-saas.corp.internal`. Same for any auxiliary services like databases. Same for internal service to service communication if your app has multiple services.

Audit your code and make sure it doesn't assume a particular service is available at a hard-coded hostname. All hostnames should be somehow injectable from the environment -- configuration files, environment variables, etc.

## Conclusion

These are just a few of the assumptions that can creep in when you're only developing for a SaaS environment that you have full control over. The common thread running through them is that they couple the business logic of your software (what your buyer is paying for) with the environment that it happens to run in today.

And even if you're not planning to deploy on-prem any time soon, de-coupling these two will lead to more robust code that will help even a purely SaaS deployment. It's a great exercise to really sit down and think critically about the actual requirements of your application, and how you could make it more portable.
