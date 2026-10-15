# Intel Linux environment (Debian 11, Python 3.8) for the original WocaR-PPO dependencies.
# Build from this directory:
#   docker build --platform linux/amd64 -t wocar-ppo .
# Open a shell with the repo mounted:
#   docker run --platform linux/amd64 -it --rm -v "$PWD:/opt/wocar-ppo" -w /opt/wocar-ppo/src wocar-ppo

FROM python:3.8-bullseye

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    MUJOCO_PY_MUJOCO_PATH=/root/.mujoco/mjpro150 \
    MUJOCO_PY_MJKEY_PATH=/root/.mujoco/mjkey.txt \
    LD_LIBRARY_PATH=/root/.mujoco/mjpro150/bin \
    MUJOCO_GL=osmesa

# Bullseye security mirrors 404 on current Debian; pin the snapshot this image was built from.
RUN printf '%s\n' \
        'deb http://snapshot.debian.org/archive/debian/20240926T000000Z bullseye main' \
        'deb http://snapshot.debian.org/archive/debian-security/20240926T000000Z bullseye-security main' \
        'deb http://snapshot.debian.org/archive/debian/20240926T000000Z bullseye-updates main' \
        > /etc/apt/sources.list \
    && printf 'Acquire::Check-Valid-Until "false";\n' > /etc/apt/apt.conf.d/99no-check-valid \
    && apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        patchelf \
        curl \
        ca-certificates \
        git \
        unzip \
        pkg-config \
        libosmesa6-dev \
        libgl1-mesa-dev \
        libglfw3 \
        libglew-dev \
        libhdf5-dev \
        libjpeg-dev \
        zlib1g-dev \
    && rm -rf /var/lib/apt/lists/* \
    && python -m pip install --upgrade "pip==24.3.1" "setuptools==69.5.1" "wheel==0.44.0"

RUN mkdir -p /root/.mujoco \
    && curl -fL --retry 3 -o /tmp/mjpro150.zip https://www.roboti.us/download/mjpro150_linux.zip \
    && unzip /tmp/mjpro150.zip -d /root/.mujoco \
    && curl -fL --retry 3 -o /root/.mujoco/mjkey.txt https://www.roboti.us/file/mjkey.txt \
    && rm /tmp/mjpro150.zip \
    && test -f /root/.mujoco/mjpro150/bin/libmujoco150.so \
    && test -s /root/.mujoco/mjkey.txt

WORKDIR /opt/wocar-ppo

COPY requirements.txt /tmp/requirements.txt
COPY docker-constraints.txt /tmp/constraints.txt

RUN python -m pip install "setuptools==65.5.1" \
    && python -m pip install --only-binary=:all: -c /tmp/constraints.txt \
        "numpy==1.19.5" \
        "Cython==0.29.36" \
        "torch==1.6.0" \
        "scipy==1.5.4" \
        "pandas==1.1.5" \
        "matplotlib==3.3.4" \
        "Pillow==7.2.0" \
        "scikit-learn==0.24.2" \
        "tables==3.7.0" \
    && python -m pip install --only-binary=:all: -c /tmp/constraints.txt \
        "cffi==1.15.1" "imageio==2.9.0" "glfw==1.12.0" \
    && python -m pip install -c /tmp/constraints.txt "lockfile==0.12.2" \
    && python -m pip install --no-build-isolation -c /tmp/constraints.txt "mujoco-py==1.50.1.68" \
    && sed -i 's/^gym$/gym==0.23.1/' /tmp/requirements.txt \
    && python -m pip install -c /tmp/constraints.txt -r /tmp/requirements.txt \
    && python - <<'PY'
import gym
import torch
import mujoco_py
env = gym.make("Hopper-v2")
obs = env.reset()
print("torch", torch.__version__)
print("mujoco_py", mujoco_py.__version__)
print("gym", gym.__version__)
print("hopper_obs", len(obs))
PY

COPY . /opt/wocar-ppo
WORKDIR /opt/wocar-ppo/src

CMD ["bash"]
