pipeline {
    agent any
    
    environment {
        NEXUS_URL = '192.168.1.13'
        NEXUS_GROUP_PORT = '8084'
    }
    
    stages {
        stage('Verificar entorno') {
            steps {
                script {
                    sh '''
                        echo "=== Entorno Jenkins ==="
                        echo "Usuario: $(whoami)"
                        echo "UID: $(id -u)"
                        echo "Directorio: $(pwd)"
                        echo "Sistema: $(uname -a)"
                        cat /etc/os-release || echo "No hay os-release"
                    '''
                }
            }
        }
        
        stage('Instalar Docker (sin sudo)') {
            steps {
                script {
                    sh '''
                        echo "=== Verificando Docker ==="
                        
                        # Verificar si ya tenemos docker
                        if command -v docker >/dev/null 2>&1; then
                            echo "✓ Docker ya está instalado"
                            docker --version
                        else
                            echo "✗ Docker no encontrado"
                            echo "Como estamos en contenedor, necesitamos usar Docker-in-Docker o socket mounting"
                            
                            # Alternativa: instalar docker client solo
                            echo "Instalando docker client..."
                            
                            # Para Alpine (si es el caso)
                            if [ -f /etc/alpine-release ]; then
                                apk update
                                apk add docker-cli
                            # Para Debian/Ubuntu
                            elif [ -f /etc/debian_version ]; then
                                apt-get update
                                apt-get install -y docker.io
                            else
                                echo "Sistema no soportado para instalación automática"
                                exit 1
                            fi
                        fi
                    '''
                }
            }
        }
        
        stage('Usar Docker socket') {
            steps {
                script {
                    sh '''
                        echo "=== Configurando Docker ==="
                        
                        # Verificar si tenemos acceso al socket de Docker
                        if [ -S /var/run/docker.sock ]; then
                            echo "✓ Socket de Docker encontrado"
                            ls -la /var/run/docker.sock
                            
                            # Configurar permisos (si es necesario)
                            DOCKER_GID=$(stat -c '%g' /var/run/docker.sock 2>/dev/null || echo "1000")
                            echo "GID del socket: $DOCKER_GID"
                            
                            # Agregar usuario jenkins al grupo docker si existe
                            if getent group docker >/dev/null; then
                                echo "Grupo docker existe"
                                id
                            fi
                        else
                            echo "✗ No hay socket de Docker"
                            echo "INFO: Jenkins necesita acceso al socket de Docker para ejecutar comandos"
                            echo "Soluciones:"
                            echo "1. Montar volumen: -v /var/run/docker.sock:/var/run/docker.sock"
                            echo "2. Usar Docker-in-Docker"
                            exit 1
                        fi
                    '''
                }
            }
