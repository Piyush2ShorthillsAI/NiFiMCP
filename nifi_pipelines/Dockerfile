# Use Ubuntu base image
FROM ubuntu:22.04

# Install minimal dependencies needed for NiFi installation
RUN apt-get update && apt-get install -y \
    wget \
    curl \
    unzip \
    openjdk-21-jdk \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Copy all contents of nifi_pipelines, including nifi_setup and pipeline subfolders
COPY . /nifi_pipelines/

# Ensure pipeline setup script is executable
RUN chmod +x /nifi_pipelines/nifi_setup/nifi_pipeline_setup.sh

# Set Java environment
ENV JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
ENV PATH=$JAVA_HOME/bin:$PATH

# Fix sudo issue by creating a sudo wrapper (since we're running as root)
RUN echo '#!/bin/bash\n\
# Sudo wrapper for Docker container (we are already root)\n\
if [ "$1" = "-E" ]; then\n\
    shift\n\
fi\n\
exec "$@"' > /usr/local/bin/sudo && \
    chmod +x /usr/local/bin/sudo

# Install NiFi but don't start it during build
RUN bash /nifi_pipelines/nifi_setup/run_all_nifi_setup.sh

# Create compatibility symlinks for standard NiFi paths
RUN mkdir -p /opt/nifi && \
    ln -sf /nifi_pipelines/nifi_setup/nifi-2.4.0 /opt/nifi/nifi-current && \
    mkdir -p /opt/nifi/nifi-current/data && \
    chmod 755 /opt/nifi/nifi-current/data && \
    echo "✅ Created compatibility symlinks for standard NiFi paths"

# Expose NiFi web interface port
EXPOSE 8443

# Create startup script
RUN echo '#!/bin/bash\n\
set -e\n\
echo "🚀 Starting NiFi..."\n\
echo "📍 Using NiFi path: /nifi_pipelines/nifi_setup/nifi-2.4.0/bin"\n\
echo "🔐 JAVA_HOME: $JAVA_HOME"\n\
echo "🔐 NIFI_HOME: /nifi_pipelines/nifi_setup/nifi-2.4.0"\n\
\n\
# Check if NiFi is already running\n\
if [ -f "/nifi_pipelines/nifi_setup/nifi-2.4.0/bin/nifi.sh" ]; then\n\
    echo "✅ NiFi binary found"\n\
    cd /nifi_pipelines/nifi_setup/nifi-2.4.0/bin\n\
    ./nifi.sh start\n\
    echo "⏳ Waiting for NiFi to start..."\n\
    sleep 30\n\
    \n\
    # Check if NiFi is actually running\n\
    if ./nifi.sh status | grep -q "Status: UP"; then\n\
        echo "✅ NiFi is running successfully"\n\
        echo "✅ NiFi should be available at https://localhost:8443/nifi"\n\
        \n\
        # Wait a bit more for NiFi to be fully ready\n\
        echo "⏳ Waiting for NiFi to be fully ready..."\n\
        sleep 15\n\
        \n\
        echo "🔧 Running pipeline setup for all pipelines in /nifi_pipelines/..."\n\
        cd /nifi_pipelines/nifi_setup\n\
        \n\
        # Retry pipeline setup up to 3 times\n\
        for attempt in 1 2 3; do\n\
            echo "🔄 Pipeline setup attempt $attempt/3..."\n\
            if bash nifi_pipeline_setup.sh; then\n\
                echo "✅ Pipeline setup completed successfully!"\n\
                break\n\
            else\n\
                echo "⚠️  Pipeline setup attempt $attempt failed, waiting 10 seconds..."\n\
                if [ $attempt -lt 3 ]; then\n\
                    sleep 10\n\
                else\n\
                    echo "❌ Pipeline setup failed after 3 attempts"\n\
                    echo "⚠️  NiFi is running but pipeline setup failed - check logs"\n\
                fi\n\
            fi\n\
        done\n\
        \n\
        echo "🎉 NiFi container is ready!"\n\
    else\n\
        echo "❌ NiFi failed to start"\n\
        ./nifi.sh status\n\
        exit 1\n\
    fi\n\
else\n\
    echo "❌ NiFi binary not found at /nifi_pipelines/nifi_setup/nifi-2.4.0/bin/nifi.sh"\n\
    ls -la /nifi_pipelines/nifi_setup/nifi-2.4.0/bin/\n\
    exit 1\n\
fi\n\
\n\
tail -f /dev/null' > /start.sh && chmod +x /start.sh

# Start NiFi when container runs
CMD ["/start.sh"]
