# Test Semaphore CI

Testing [Semaphore CI](https://semaphoreci.com) to recreate [Amazon-internal Pipelines](https://aws.amazon.com/builders-library/cicd-pipeline) ([video](https://www.youtube.com/watch?v=ngnMj1zbMPY)).

## Issues

### Automatic Rollbacks

* Rolling back to an old deployment generally requires knowing the last successful deployment workflow ([API](https://docs.semaphoreci.com/reference/api-v1alpha/#retrieving-deployment-history)) and triggering its rollback promotion ([API](https://docs.semaphoreci.com/reference/api-v1alpha/#triggering-a-promotion)).
	* Needed since the deploy commands might be different between the old and new deployment pipelines.
		* [semaphoreci-demos/semaphore-demo-cicd-kubernetes](https://github.com/semaphoreci-demos/semaphore-demo-cicd-kubernetes) only shows how to rollback a canary deployment. We're looking for a late-stage full rollback.
	* Since the rollback pipeline should share the same [pipeline queue](https://docs.semaphoreci.com/essentials/pipeline-queues) as the rollforward pipeline, a deadlock occurs if the rollforward pipeline waits on rollback pipeline completion.
* [Fast-fail](https://docs.semaphoreci.com/essentials/fail-fast-stop-running-tests-on-the-first-failure) is desired to terminate deployment commit (test + monitor) jobs, so a `result = 'failed'` promotion to a rollback pipeline is needed. Unfortunately, this might have some undesirable consequences when combined with [auto-cancel](https://docs.semaphoreci.com/essentials/auto-cancel-previous-pipelines-on-a-new-push). For example:
	* Auto-cancel: Running | Queued
		* Time 1:
			* 🔵 Rollforward A
		* Time 2:
			* 🔴 Rollforward A
			* 🟠 Rollback A
		* Time 3:
			* 🔴 Rollforward A
			* 🟠 Rollback A
			* 🟠 Rollforward B
		* Time 4:
			* 🔴 Rollforward A
			* ⚪️ Rollback A ← ⛔️
			* 🔵 Rollforward B ← ⛔️
	* Auto-cancel: None
		* Time 1:
			* 🔵 Rollforward A
		* Time 2:
			* 🔵 Rollforward A
			* 🟠 Rollforward B
		* Time 3:
			* 🔴 Rollforward A
			* 🟠 Rollforward B
			* 🟠 Rollback A
		* Time 4:
			* 🔴 Rollforward A
			* 🔵 Rollforward B ← ⛔️
			* 🟠 Rollback A

There's no built-in auto-rollback feature.

Deployment transactions essentially aren't a first-class citizen. The only practical option is to not auto-rollback. Users either:

1. Fix the issue and rollforward.
2. Revert `main` and rollforward.

### Blocking Promotions

#### Time Window Blockers

Doesn't seem possible.

#### Emergent Blockers

Global emergent blockers can be added by [pausing projects](https://docs.semaphoreci.com/essentials/project-workflow-trigger-options#pausing-a-project).

Unfortunately, this doesn't create any workflows instead of creating them but leaving them queued.

Deployment target specific blockers can be added by [deactivating deployment targets](https://docs.semaphoreci.com/essentials/deployment-targets).

Unfortunately, this fails automatic promotions instead of queueing them so they resume when the deployment target is reactivated.

### Emergency Deployments

Emergency rollbacks and rollforwards are possible by manually running a promotion.

Pipelines started with a manual promotion, however, will trigger all subsequent promotions. For rollforwards, this probably isn't desirable if this is an ad-hoc deployment to a deployment target for testing (e.g. staging). For rollbacks, the rollback pipeline should have no promotion.
