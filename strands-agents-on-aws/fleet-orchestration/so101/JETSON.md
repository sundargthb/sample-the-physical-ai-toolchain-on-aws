# Running an SO-101 robot host on an NVIDIA Jetson Orin Nano

The robot host is the same code as on a laptop (`so101/arm_host.py`). Only the machine changes. The follower's USB
and its cameras plug into the Jetson, the host connects to AWS IoT Core from the Jetson, and the ACT pick policy runs
on the Jetson's GPU. The planner on AgentCore Runtime doesn't change.

## 1. Python and PyTorch

LeRobot needs Python 3.12. JetPack ships its own Python, and NVIDIA's PyTorch builds for Jetson match that Python. So
use a container that already pairs a CUDA-enabled PyTorch with Python 3.12 (an NVIDIA or Jetson AI Lab image for your
JetPack version). Don't mix pip PyTorch with the system Python. Check inside the environment you'll run the host from:

    cat /etc/nv_tegra_release
    python3 -c "import sys, torch; print(sys.version, torch.__version__, torch.cuda.is_available())"

`torch.cuda.is_available()` must print True.

## 2. Install the host's dependencies

    pip install "lerobot[feetech]==0.6.1" awsiotsdk boto3 cryptography huggingface_hub

The host uses boto3 once, at start-up, to look up your account's IoT Core endpoint. Export `IOT_ENDPOINT` to skip
that call. Copy `fleet/` and `so101/` (with your `arms.json` and `fleet/operator_pub.hex`) to the Jetson.

## 3. Hardware

- Plug the follower's USB and its cameras into the Jetson. Give your user the serial port
  (`sudo usermod -aG dialout $USER`, then log in again).
- Copy the arm's calibration file from the machine you calibrated on to the same path on the Jetson
  (`~/.cache/huggingface/lerobot/calibration/robots/so_follower/<calibration_id>.json`).
- Camera indices differ on Linux: list them with `v4l2-ctl --list-devices` and update `arms.json`.

## 4. Credentials

Give the arm its own X.509 certificate for its IoT thing, or export short-lived IAM credentials for the session.
Never put keys on a command line.

## 5. Preflight, then serve

    ARM=arm-1 python so101/arm_host.py --preflight     # reads motors and cameras, zero writes
    ARM=arm-1 python so101/arm_host.py                 # the policy picks cuda automatically

The host runs the policy at 30 Hz, so each control step must stay under 33 ms.
