# 以輕量 Ubuntu 22.04 為基底
FROM mcr.microsoft.com/devcontainers/base:ubuntu-22.04

ARG TOOLCHAIN_URL=https://developer.arm.com/-/media/Files/downloads/gnu/13.2.rel1/binrel/arm-gnu-toolchain-13.2.rel1-x86_64-arm-none-eabi.tar.xz

# 安裝編譯、終端與燒錄所需工具
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        build-essential make wget python3 python3-pip minicom lrzsz ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# 下載並安裝 GNU Arm Toolchain 13.2 (arm-none-eabi)
RUN wget -q ${TOOLCHAIN_URL} -O /tmp/toolchain.tar.xz && \
    mkdir /opt/arm-toolchain && \
    tar -xf /tmp/toolchain.tar.xz -C /opt/arm-toolchain --strip-components=1 && \
    rm /tmp/toolchain.tar.xz
ENV PATH="/opt/arm-toolchain/bin:${PATH}"

# 安裝 Python xmodem 及其他燒錄相依套件
RUN pip install --no-cache-dir xmodem pyserial

# 安裝 Node.js 18.x 與 npm
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - && \
    apt-get install -y nodejs && \
    rm -rf /var/lib/apt/lists/*

# 全域安裝 Claude Code CLI
RUN npm install -g --unsafe-perm @anthropic-ai/claude-code && \
    npm install -g @google/gemini-cli

# install Rovo dev CLI
RUN curl -LO "https://acli.atlassian.com/linux/latest/acli_linux_arm64/acli" && \
    chmod +x acli && \
    install -m 0755 acli /usr/local/bin/acli

# 預設工作目錄即為 /workspace 以符合 Dev Containers 約定
WORKDIR /workspace

# 在 devcontainer 環境中，VS Code 會處理容器的生命週期
# 我們不需要 CMD，VS Code 會覆蓋它
