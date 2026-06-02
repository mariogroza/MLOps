pipeline {
    agent any

    stages {

        stage('Cleanup podman stale state') {
            steps {
                // After a reboot, podman rootless may have stale cache dirs that need to be cleared
                sh '''
                    STORAGE_RUN=$(ls /tmp/ | grep storage-run)
                    if [ -n "$STORAGE_RUN" ]; then
                        rm -rf /tmp/${STORAGE_RUN}/containers
                        rm -rf /tmp/${STORAGE_RUN}/libpod/tmp
                        echo "Cleaned up stale podman state from /tmp/${STORAGE_RUN}"
                    else
                        echo "No stale podman state found, skipping..."
                    fi
                '''
            }
        }

        stage('Create /opt/models directory') {
            steps {
                // Run mkdir as root via sudo since /opt is owned by the system
                // Then change ownership to jenkins so the model file can be written without root
                sh 'sudo mkdir -p /opt/models'
                sh 'sudo chown jenkins:jenkins /opt/models'
            }
        }

        stage('Create podman network') {
            steps {
                sh '''
                    # Check if the network already exists to avoid errors on re-runs
                    if sudo podman network exists llm-net 2>/dev/null; then
                        echo "Network llm-net already exists, skipping..."
                    else
                        # Create an isolated virtual network for container-to-container communication
                        sudo podman network create llm-net
                        echo "Network llm-net created."
                    fi
                '''
            }
        }

        stage('Download model') {
            steps {
                sh '''
                    # Skip download if the model file is already present
                    if [ -f /opt/models/model.gguf ]; then
                        echo "Model already exists, skipping download..."
                    else
                        echo "Downloading model..."
                        # Download the Qwen2.5 0.5B Instruct model in Q4_K_M quantized GGUF format from Hugging Face
                        curl -L --progress-bar \
                             -o /opt/models/model.gguf \
                             "https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q4_k_m.gguf"
                        echo "Model downloaded: $(du -sh /opt/models/model.gguf)"
                    fi
                '''
            }
        }

        stage('Start llama-server container') {
            steps {
                sh '''
                    # Remove existing container if present (e.g. from a previous pipeline run)
                    if sudo podman ps -a --format "{{.Names}}" | grep -q "^llama-server$"; then
                        echo "Container llama-server already exists, removing..."
                        sudo podman rm -f llama-server
                    fi

                    # Start the llama.cpp inference server and mount the models directory
                    # :Z flag is required for SELinux systems (e.g. Fedora/RHEL) to relabel the volume
                    # The server listens on port 8080 inside the container, mapped to 8080 on the host
                    sudo podman run -d \
                        --name llama-server \
                        --network llm-net \
                        -v /opt/models:/models:Z \
                        -p 8080:8080 \
                        ghcr.io/ggml-org/llama.cpp:server \
                        -m /models/model.gguf --host 0.0.0.0 --port 8080

                    # Give the server a few seconds to initialize before starting open-webui
                    echo "Waiting for llama-server to be ready..."
                    sleep 10
                '''
            }
        }

        stage('Start open-webui container') {
            steps {
                sh '''
                    # Remove existing container if present (e.g. from a previous pipeline run)
                    if sudo podman ps -a --format "{{.Names}}" | grep -q "^open-webui$"; then
                        echo "Container open-webui already exists, removing..."
                        sudo podman rm -f open-webui
                    fi

                    # Start the Open WebUI frontend, exposed on host port 3000
                    # OPENAI_API_BASE_URL points to llama-server using the internal llm-net network on port 8080
                    sudo podman run -d \
                        --name open-webui \
                        --network llm-net \
                        -p 3000:8080 \
                        -e OPENAI_API_BASE_URL=http://llama-server:8080/v1 \
                        -e OPENAI_API_KEY=sk-no-key-required \
                        ghcr.io/open-webui/open-webui:main

                    echo "Waiting for open-webui to start..."
                    sleep 5
                '''
            }
        }

        stage('Verify containers are running') {
            steps {
                sh '''
                    echo "=== Running containers ==="
                    sudo podman ps

                    echo ""
                    echo "=== Checking llama-server ==="
                    # Fail the pipeline if llama-server is not in the running containers list
                    if sudo podman ps --format "{{.Names}}" | grep -q "^llama-server$"; then
                        echo "llama-server is running"
                    else
                        echo "llama-server is NOT running"
                        # Print logs to help diagnose the failure
                        sudo podman logs llama-server || true
                        exit 1
                    fi

                    echo ""
                    echo "=== Checking open-webui ==="
                    # Fail the pipeline if open-webui is not in the running containers list
                    if sudo podman ps --format "{{.Names}}" | grep -q "^open-webui$"; then
                        echo "open-webui is running"
                    else
                        echo "open-webui is NOT running"
                        # Print logs to help diagnose the failure
                        sudo podman logs open-webui || true
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        success {
            echo """
Deploy successful!
   - llama-server API : http://localhost:8080
   - open-webui       : http://localhost:3000
            """
        }
        failure {
            // On failure, dump container state and logs for easier debugging
            sh '''
                echo "=== Debug info ==="
                sudo podman ps -a || true
                sudo podman logs llama-server 2>/dev/null || true
                sudo podman logs open-webui 2>/dev/null || true
            '''
        }
    }
}