# Serve from a cloud VM

Read this file together with the `serve-policy` skill, in
[`skills/serve-policy/SKILL.md`](../SKILL.md). A cloud VM is a Linux
machine with a GPU that the user rents from a cloud, such as an AWS EC2
instance or a GCP Compute Engine instance.

## Run the skill on the VM

The user connects to the VM over SSH and runs the agent there. Follow
`skills/serve-policy/SKILL.md` from Phase 1, with the VM as this machine.

- `manifold auth login` prints a link. The user opens the link in a
  browser on their laptop.
- Start the serve command in a tmux session, as the "Serve" step in
  `skills/serve-policy/SKILL.md` describes. The serve command then keeps
  running when the SSH session closes.
- The VM needs outbound network access only. `manifold policy serve`
  connects out to Manifold, and to a relay server that passes traffic
  between the VM and the benchmark. The default firewall rules on AWS
  and GCP allow this traffic.

## Pick an image with the NVIDIA driver

`nvidia-smi` may fail on the VM outside the agent's sandbox. Then the VM
image lacks the NVIDIA driver. Ask the user to create the VM from an
image that includes the driver:

| Cloud | Image |
|---|---|
| AWS | A Deep Learning AMI. The image includes the driver. |
| GCP | A Deep Learning VM image. The image asks to install the driver when the user first logs in. The user answers yes. |

## If the cloud interrupts the VM

A spot VM can end during a run. AWS gives a spot VM 2 minutes of
notice, and GCP gives 30 seconds. The serve command may stop before it
removes its registration.

After an interruption, do these steps in order:

1. Run `manifold run get <run-id>`. Manifold waits about 90 seconds for
   heartbeats from the VM. Then it puts the run back in the queue and
   deletes the episodes that the run recorded.
2. Revoke the VM's runner, as the "Stop serving" step in
   `skills/serve-policy/SKILL.md` describes. The runner has the VM's
   hostname as its name. For about 5 minutes after the VM ends, Manifold
   can still place the run on that runner.
3. Start the VM again, or create a new VM, and serve the same version.
   The queued run waits for that version.
4. Tell the user that the run starts again from its first episode.
