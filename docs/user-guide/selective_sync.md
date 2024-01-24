# 선택적 동기화 (Selective Sync)

*선택적 동기화*는 동기화할 리소스를 말합니다. 어떤 리소스를 동기화할지 UI에서 선택할 수 있습니다. is one where only some resources are sync'd. You can choose which resources from the UI:

![selective sync](../assets/selective-sync.png)

선택적 동기화를 할 때, 주의사항입니다.

* 동기화는 히스토리에 기록되지 않기 때문에, 롤백이 불가능합니다.
* 훅이 동작하지 않습니다.

## 선택적 동기화의 옵션

>v1.8

Turning on selective sync option which will sync only out-of-sync resources.
See [sync options](sync-options.md#selective-sync) documentation for more details.
