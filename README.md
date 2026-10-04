# Terraform Practical - Exercise 3

This repo is a **timed, hands-on Terraform exercise** (30 minutes), in the same
style as the real interview task: **Terraform + GitLab + VS Code**.

This one is about **modules** (reusable building blocks) and **high availability**
(keeping things running even if one data centre has a problem).

> There is **no solution in the main repo on purpose** - you work through it
> yourself. (A fully-worked, heavily-commented solution lives in the `solution/`
> folder if you get stuck or want to check your answer afterwards.)

---

## 0. The two ideas this exercise teaches (read this first)

**What is a module?**
A module is just a **folder of Terraform code you can reuse**. Instead of
copy-pasting the same VPC/subnets/firewall for every environment, you write it
**once** inside a module, then "call" that module as many times as you like with
different settings (e.g. one call for `nonprod`, one for `prod`). This is how
teams get **speed and consistency** - the same tested building block everywhere.

Think of a module like a **recipe**. You write the recipe once, then cook it
twice with different ingredients (different IP ranges) to get two different meals
(two environments).

**What is "high availability" (HA) here?**
Private servers usually can't reach the internet directly - they go out through a
**NAT gateway**. If you build only **one** NAT gateway and the data centre
(availability zone) it lives in has an outage, **all** your private servers lose
internet access. The HA fix is to build **one NAT gateway per zone**, so a problem
in one zone doesn't take down the others.

---

## 1. What the exercise is (the brief)

> "Here's a working landing-zone network for **one** environment. Right now it's
> all in a single `main.tf`. We need it as a **reusable module** so we can build
> both **nonprod** and **prod** from the same code, and we need
> **high-availability internet access** for the private subnets (a NAT gateway per
> zone). You've got **30 minutes**."
>
> - Turn the network code into a **module** at `terraform/modules/network`.
> - **Call the module twice**: once for `nonprod`, once for `prod`, with different
>   IP ranges.
> - Add **HA NAT** (one NAT gateway per availability zone) that can be switched
>   on or off with a variable.
> - Raise a **merge request** with your changes.

You will **not** have live AWS credentials, so focus on **`init`, `fmt` and
`validate`**, and *say out loud* what you'd expect a real `plan` to show.

### The rules on the day
- **Time:** 30 minutes. You'll be told at 15, 25 and 28 minutes.
- **Tools:** Terraform CLI + Git. **No** live cloud credentials.
- **Narrate:** talk through your reasoning the whole time - *silence scores nothing*.
- **Deliver:** a feature branch, small meaningful commits, and a merge request.

---

## 2. One-time setup (Windows)

### 2a. Git (if `git --version` fails)
Install from <https://git-scm.com/download/win> and accept all defaults.

### 2b. Terraform
```powershell
winget install HashiCorp.Terraform
```
Then **close and reopen PowerShell**.

### 2c. Check both tools work
```powershell
git --version
terraform version
```

---

## 3. Get the exercise onto your machine

```powershell
git clone https://github.com/akabii/terraform-exercise-3.git
cd terraform-exercise-3
```
The code lives in the **`landing-zone/`** folder.

---

## 4. How to do it, step by step (narrate every step out loud)

### Step 1 - Look before you touch
```powershell
cd landing-zone\terraform
```
Read `main.tf`. Notice it builds a VPC, public + private subnets, an internet
gateway, and a public route table - all for a **single** environment.

### Step 2 - Get a clean baseline
```powershell
terraform init -backend=false
terraform fmt -check -recursive
terraform validate     # should say: Success!
```
This proves the starting point works before you change anything.

### Step 3 - Make a feature branch
```powershell
git checkout -b feature/network-module
```

### Step 4 - Build the module
Create a folder `terraform/modules/network` and move the network resources into
it. A module is usually three small files:
- `variables.tf` - the settings the module accepts (environment, vpc_cidr, azs, enable_nat)
- `main.tf` - the resources (what the original main.tf had)
- `outputs.tf` - values the module hands back (e.g. the VPC id, subnet ids)

Tip: inside the module, use `var.environment`, `var.vpc_cidr`, etc. - don't
hard-code anything.

### Step 5 - Call the module twice from the root `main.tf`
Replace the root `main.tf` resources with two `module` blocks:
```hcl
module "nonprod" {
  source      = "./modules/network"
  environment = "nonprod"
  vpc_cidr    = "10.60.0.0/16"
  azs         = var.azs
  enable_nat  = false
}

module "prod" {
  source      = "./modules/network"
  environment = "prod"
  vpc_cidr    = "10.61.0.0/16"
  azs         = var.azs
  enable_nat  = true
}
```
Run `terraform init` again (new modules must be initialised), then `validate`.

### Step 6 - Add HA NAT inside the module
Add, controlled by `var.enable_nat`:
- one Elastic IP (`aws_eip`) per zone,
- one NAT gateway (`aws_nat_gateway`) per zone, each in that zone's public subnet,
- one private route table per zone with a default route to that zone's NAT,
- associate each private subnet with its zone's private route table.

### Step 7 - Prove it's clean, then push and open a merge request
```powershell
terraform fmt -recursive
terraform validate
git push -u origin feature/network-module
```
Open a merge request with a clear description of what you changed and why.

---

## 5. What you're being marked on

- A clean baseline first (`init` / `validate` before changing anything).
- A **real module** with sensible inputs and outputs (nothing hard-coded inside).
- The module **called twice** for two environments with different IP ranges.
- **HA NAT**: one NAT gateway per zone, and being able to explain *why* that's more
  resilient than a single shared NAT.
- Feature branch, small commits, and a merge request with a real description.
- Being able to **explain every choice** in simple terms.

---

## 6. Golden rules for the day
- **Talk constantly.** Explain what you're reading, thinking, and doing.
- **Baseline first.** Run `init`/`validate` before editing.
- **Don't hard-code inside a module.** Everything variable goes through inputs.
- **Small commits, clear messages.**
- **Manage the clock.** A tidy, pushed, partly-finished answer beats a rushed mess.
- **It's fine not to finish.** Method beats completion.

Good luck - think out loud and back yourself.
