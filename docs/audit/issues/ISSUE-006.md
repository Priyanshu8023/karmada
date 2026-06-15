# ISSUE-006: Security — gRPC Insecure Skip Verify Option

## Summary
The scheduler estimator gRPC connection supports `InsecureSkipServerVerify`, allowing TLS bypass vulnerable to MITM attacks.

## Suggested Patch
```diff
 func WithSchedulerEstimatorConnection(...) Option {
     return func(o *schedulerOptions) {
+        if insecureSkipVerify {
+            klog.Warning("Scheduler estimator TLS verification is disabled. This is insecure and should not be used in production.")
+        }
         o.schedulerEstimatorClientConfig = &grpcconnection.ClientConfig{
             InsecureSkipServerVerify: insecureSkipVerify,
             ...
         }
     }
 }
```

## Validation Checklist
- [ ] Warning log added
- [ ] Documentation updated
- [ ] CI passed
