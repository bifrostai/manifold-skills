# manifold-sdk import paths

- `manifold.recipes`. `read_pairing`, `launch_server`, `serve`, `evaluate`,
  `run_benchmark`, `run_sharded_benchmark`, `run_episodes`, `write_rollup`,
  `OpenLoopChunkQueue`, `ChunkEndpoint`, `PolicyProfile`, `resolve`,
  `from_lerobot_checkpoint` / `SignatureSuggestion`, `describe`, `Recorder`,
  `dump`, `load`, `NO_RECORDER`
- `manifold.recipes.serving`. `PolicyEndpoint` and `Session` protocols (NOT
  re-exported by `manifold.recipes`)
- `manifold.core.check`. `check_compatibility` returns `Report`
- `manifold.core.verify`. `verify` returns `VerifyReport`
- `manifold.core.pipeline`. `Pipeline`
- `manifold.core.policy`. `PolicySignature`
- `manifold.core.embodiment`. `EEActionSpace`, `JointActionSpace`,
  `UnifiedActionSpace`, `Proprioception`, `EEObservationSpec`,
  `GripperObservationSpec`
- `manifold.core.conventions`. `RotationFormat`, `GripperFormat`, `Frame`
  (pass enum members, never string values)
- `manifold.core.native_layout`. `NativeLayout`, `LayoutEntry`
  (`.from_camera` / `.from_state` / `.from_instruction`), `SourceKind`,
  `Slice`, `Split`, `BatchAxis`, `DtypeCast`, `Component`, `Assemble`
- `manifold.benchmarks`. `ALL`, `LIBERO`, `SIMPLER`, `ROBOCASA`
- `manifold.adapters`. `PackToNativeLayout`, `UnpackFromNativeLayout`,
  `ObservationTap`, `ActionTap` (convention adapters are one level down)
- `manifold.adapters.observation`. `ProprioRotationAdapter`,
  `FrameRebaseAdapter`, `DynamicFrameRebaseAdapter`, `ObservedGripperAdapter`,
  `ResizeCameras`, `Rotate180Cameras`, `FlipVerticalCameras`,
  `SwapChannelOrder`, `StackFrameHistory`
- `manifold.adapters.action`. `RotationFormatAdapter`,
  `GripperPolarityAdapter`, `GripperThresholdAdapter`,
  `UnifiedGripperThresholdAdapter`, `UnifiedSliceAdapter`, `BasePinWiden`,
  `DiscreteBinarize`
- `manifold.lib.rotation.convert`. Driver-side rotation re-encode
- `manifold.lib.gripper`. Action-side gripper remaps
