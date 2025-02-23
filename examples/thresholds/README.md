# Cost Thresholds Example

This example shows how to set thresholds that limit when a comment is posted. For simplicity, this is based off the terraform-plan-json example, which does not require Terraform to be installed. A similar thing can be done with bash and [jq](https://stedolan.github.io/jq/).

[//]: <> (BEGIN EXAMPLE)
```yml
name: Thresholds
on: [pull_request]

jobs:
  thresholds:
    name: Thresholds
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Infracost
        uses: infracost/actions/setup@v1
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Run Infracost
        run: infracost breakdown --path=examples/thresholds/code/plan.json --format=json --out-file=/tmp/infracost.json
        
      - name: Calculate Cost Change
        id: cost-change
        uses: actions/github-script@v5
        with:
          script: |
            // Read the breakdown JSON and get costs
            const breakdown = require('/tmp/infracost.json');
            const past = breakdown.pastTotalMonthlyCost;
            const current = breakdown.totalMonthlyCost;
            const costChange = breakdown.diffTotalMonthlyCost;
            const absoluteCostChange = Math.abs(costChange);

            // Calculate the percent change
            let percentChange = 999; // default to a high number so we post a comment if there's no past cost
            if (past !== "0") {
              percentChange = 100 * ((current - past) / past);
            }
            const absolutePercentChange = Math.abs(percentChange);

            console.log(`past: ${past}`);
            console.log(`current: ${current}`);
            console.log(`cost-change: ${costChange}`);
            console.log(`absolute-cost-change: ${absoluteCostChange}`);
            console.log(`percent-change: ${percentChange}`);
            console.log(`absolute-percent-change: ${absolutePercentChange}`);

            // Set the calculated diffs as outputs to be used in future steps
            core.setOutput('absolute-percent-change', absolutePercentChange);
            core.setOutput('percent-change', percentChange);
            core.setOutput('absolute-cost-change', absoluteCostChange );
            core.setOutput('cost-change', costChange);

      - name: Post the comment
        uses: infracost/actions/comment@v1
        if: ${{ steps.cost-change.outputs.absolute-percent-change > 1 }} # Only comment if cost changed by more than plus or minus 1%
        # if: ${{ steps.cost-change.outputs.percent-change > 1 }} # Only comment if cost increased by more than 1%
        # if: ${{ steps.cost-change.outputs.absolute-cost-change > 100 }} # Only comment if cost changed by more than plus or minus $100
        # if: ${{ steps.cost-change.outputs.cost-change > 100 }} # Only comment if cost increased by more than $100
        with:
          path: /tmp/infracost.json
          behavior: update # Create a single comment and update it. See https://github.com/infracost/actions/tree/master/comment for other options
```
[//]: <> (END EXAMPLE)
