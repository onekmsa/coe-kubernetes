#### pods가 삭제되지 않고 계속 남아 있음
- 현상: kubectl delete pod --all 로 모든 pod 삭제를 해보려 했으나, deleted 메시지만 남고 실제 삭제 되지 않음.  
- 해결: 해당 pod을 강제 종료 함  
```ssh
kubectl delete pod NAME --grace-period=0 --force
```
