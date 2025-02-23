# Conftest Example

This example shows how to set cost policies using [Conftest](https://www.conftest.dev/).  For simplicity, this is based off the terraform-plan-json example, which does not require Terraform to be installed.

When the policy checks pass, the GitHub Action step called "Check Conftest Policies" passes and outputs `3 tests, 3 passed, 0 warnings, 0 failures, 0 exceptions` in the action logs. When the policy checks fail, that step fails and the action logs show the details of the failing policies.

Create a policy file (e.g. `policy.rego`) that checks the Infracost JSON: 
```rego
package main

deny_totalDiff[msg] {
	maxDiff = 1500.0
	to_number(input.diffTotalMonthlyCost) >= maxDiff

	msg := sprintf(
		"Total monthly cost diff must be < $%.2f (actual diff is $%.2f)",
		[maxDiff, to_number(input.diffTotalMonthlyCost)],
	)
}

deny_instanceCost[msg] {
	r := input.projects[_].breakdown.resources[_]
	startswith(r.name, "aws_instance.")

	maxHourlyCost := 2.0
	to_number(r.hourlyCost) > maxHourlyCost

	msg := sprintf(
		"AWS instances must cost less than $%.2f\\hr (%s costs $%.2f\\hr).",
		[maxHourlyCost, r.name, to_number(r.hourlyCost)],
	)
}

deny_instanceCost[msg] {
	r := input.projects[_].breakdown.resources[_]
	startswith(r.name, "aws_instance.")

	baseHourlyCost := to_number(r.costComponents[_].hourlyCost)

	sr_cc := r.subresources[_].costComponents[_]
	sr_cc.name == "Provisioned IOPS"
	iopsHourlyCost := to_number(sr_cc.hourlyCost)

	iopsHourlyCost > baseHourlyCost

	msg := sprintf(
		"AWS instance IOPS must cost less than compute usage (%s IOPS $%.2f\\hr, usage $%.2f\\hr).",
		[r.name, iopsHourlyCost, baseHourlyCost],
	)
}
```

Then use Conftest to test infrastructure cost changes against the policy.

[//]: <> (BEGIN EXAMPLE)
```yml
name: Conftest
on: [pull_request]

jobs:
  conftest:
    name: Conftest
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Infracost
        uses: infracost/actions/setup@v1
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Setup Conftest
        uses: artis3n/setup-conftest@v0
        with:
          conftest_wrapper: false

      - name: Run Infracost
        run: infracost breakdown --path=examples/conftest/code/plan.json --format=json --out-file=/tmp/infracost.json

      - name: Check Conftest Policies
        run: conftest test --policy examples/conftest/policy /tmp/infracost.json             
```
[//]: <> (END EXAMPLE)
