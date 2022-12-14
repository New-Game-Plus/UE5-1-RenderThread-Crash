# RenderThread Crash on UE5.1 (2022-12-15)

Instructions and configurations that reproduce UE5.1 crashing with a template project while pixel streaming from a Kubernetes container.

## Dependencies

- Coreweave account/tenant
- UE5.1
- Linux machine
- kubectl
- docker
- Github account

_Note:_ It's likely possible to test this on any Kubernetes cluster with an NVIDIA GPU available (e.g. GKE).

## Steps to Reproduce

1. On Linux, create a new UE5.1 project named `ThirdPersonTemplate` using the _Third Person_ template
2. In UE Editor, enable the PixelStreaming plugin
3. Package the project from UE Editor
4. Create a volume and a throwaway container at Coreweave
```
kubectl apply -f data-copy.yaml
```
5. Copy the packaged UE5 project files to the Coreweave volume, it'll take a while.
```
kubectl cp ./Linux data-copy:/data/
```
6. Setup your local user docker config with auth to access EpicGames on Github Container Registry: https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Containers/ContainersQuickStart/
7. Create the docker credentials secret at Coreweave
```
cd ~/ && kubectl create secret generic ghcr-epicgames-docker-credentials \
   --from-file=.dockerconfigjson=./.docker/config.json \
   --type=kubernetes.io/dockerconfigjson
```
8. Create the pod, service & configmap
```
kubectl apply -f third-person-ue51.yaml
```
9. Get the public IP
```
kubectl get svc third-person-ue51
```
10. Browse to the public IP and start pixel PixelStreaming
11. Tail the log
```
kubectl logs third-person-ue51 -c unreal -f | tee -a third-person-ue51.log
```
12. Wait up to 30 minutes for the pixelstreaming to freeze and UE will crash with a log like this:
```
[2022.12.14-13.37.52:806][685]LogCore: Error: appError called: Fatal error: [File:./Runtime/RenderCore/Private/RenderingThread.cpp] [Line: 1273]
GameThread timed out waiting for RenderThread after 60.00 secs
0x000000000d26d0a0 ThirdPersonTemplate!void DispatchCheckVerify<void, GameThreadWaitForTask(TRefCountPtr<FGraphEvent> const&, ENamedThreads::Type, bool)::$_221, FLogCategoryLogRendererCore, char16_t [63], double>(GameThreadWaitForTask(TRefCountPtr<FGraphEvent> const&, ENamedThreads::Type, bool)::$_221&&, FLogCategoryLogRendererCore const&, char16_t const (&) [63], double const&)()
0x00000000095851ca ThirdPersonTemplate!FRenderCommandFence::Wait(bool) const()
0x000000000403354a ThirdPersonTemplate!FEngineLoop::Tick()()
0x000000000403885a ThirdPersonTemplate!GuardedMain(char16_t const*)()
0x000000000bbfaa98 ThirdPersonTemplate!CommonUnixMain(int, char**, int (*)(char16_t const*), void (*)())()
0x00007f33b3c86bf7 libc.so.6!__libc_start_main(+0xe6)
0x000000000402b029 ThirdPersonTemplate!_start()
```
