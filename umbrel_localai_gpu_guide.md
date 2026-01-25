# Enabling GPU Acceleration for LocalAI on UmbrelOS (RTX 3060)

This guide documents how to enable full CUDA GPU acceleration for LocalAI on UmbrelOS (Debian 13) using an NVIDIA RTX 3060 (12GB).  
The goal is to expose the GPU cleanly to Docker and then to the LocalAI container.

---

## 1. Verify the GPU works on the host

Before touching Docker, confirm the OS sees the GPU:

```bash
nvidia-smi
```

You should see your RTX 3060, driver version, and VRAM.  
If this fails, nothing above it will work.

---

## 2. Remove broken or mixed NVIDIA / CUDA installs

Umbrel is Debian-based. Mixing Ubuntu CUDA repos causes DKMS failures.

```bash
dpkg -l | egrep 'nvidia|cuda|libnvidia'
sudo apt purge -y $(dpkg -l | awk '/^ii/ && ($2 ~ /^(nvidia|cuda|libnvidia)/){print $2}')
sudo apt autoremove -y --purge
```

Confirm:

```bash
dpkg -l | egrep 'nvidia|cuda'
```

Only firmware packages should remain.

---

## 3. Install NVIDIA driver from Debian repositories

```bash
sudo apt update
sudo apt install nvidia-driver
sudo reboot
```

Verify:

```bash
nvidia-smi
ls /dev/nvidia*
```

Expected:

```
/dev/nvidia0
/dev/nvidiactl
/dev/nvidia-uvm
```

---

## 4. Install NVIDIA Container Toolkit

```bash
sudo apt install nvidia-container-toolkit
sudo systemctl restart docker
```

Verify:

```bash
sudo docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

---

## 5. Switch LocalAI to a CUDA-enabled image

Edit:

```
~/umbrel/app-data/localai/docker-compose.yml
```

```yaml
# image: localai/localai:v3.8.0@sha256:...
image: localai/localai:latest-gpu-nvidia-cuda-12
```

Pull first:

```bash
sudo docker pull localai/localai:latest-gpu-nvidia-cuda-12
```

---

## 6. Add NVIDIA environment variables

```yaml
environment:
  - NVIDIA_VISIBLE_DEVICES=all
  - NVIDIA_DRIVER_CAPABILITIES=compute,utility
  - NVIDIA_DISABLE_REQUIRE=1
  - NVIDIA_REQUIRE_CUDA=
```

---

## 7. Add Docker GPU reservation block

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

---

## 8. Restart Umbrel / LocalAI

Restart from Umbrel UI or reboot.

---

## 9. Verify CUDA inside container

```bash
sudo docker exec -it localai_api_1 sh -lc 'ldconfig -p | egrep -i "libcuda|libcudart|libcublas"'
sudo docker exec -it localai_api_1 ls -l /dev/nvidia*
```

---

## 10. Prove GPU usage

```bash
watch -n 0.5 nvidia-smi
```

Run inference in another terminal.

---

## Final Result

LocalAI is now fully GPU-accelerated on Umbrel using the RTX 3060.
