# Configuring Non-Root Security Context for Kubernetes Pods


### Overview
Implementing pod security context settings is essential for enhancing the security posture of your Kubernetes workloads. By configuring pods to run as non-root users with appropriate privileges, you can significantly reduce the attack surface and adhere to the principle of least privilege.

### Security Context Fundamentals
Pods should operate with a defined user or group rather than running as root. The securityContext configuration allows you to specify settings such as runAsUser or fsGroup to assign appropriate permissions. It is critical to allocate only the minimum required privileges and avoid using security contexts as a means to gain additional permissions.

### References:
[Configure a Security Context for a Pod or Container | Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

[Docker Security Documentation](https://docs.docker.com/engine/security/)

### Implementation Approaches
**1. API and Function Services Configuration**

The following approach applies to both API and function services with minimal differences.
- Dockerfile Configuration
  - Create a dedicated user and group with a non-privileged ID (e.g., `1000`):
   ```dockerfile
   # Create non-root user and group
   RUN groupadd -g 1000 appgroup && \
       useradd -u 1000 -g appgroup -s /bin/bash -m appuser
   ```
   - Modify ownership of application directories:
   ```dockerfile
   # Change ownership of application directory
   RUN chown -R appuser:appgroup /app
   ```
   - Configure service ports above 1024, as non-root users cannot bind to privileged ports below 1024.
   - Specify the user at the end of the Dockerfile:
   ```dockerfile
   # Run as non-root user
   USER appuser
   ```

- Kubernetes Manifest Configuration

  - Ensure consistent port configuration across all manifests (deployment, service, and ingress).<br/><br/>
  Deployment example:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: api-service
  spec:
    template:
      spec:
        securityContext:
          runAsUser: 1000
          runAsGroup: 1000
          fsGroup: 1000
        containers:
        - name: api-container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
    ```
    - Service and Ingress configuration must align with the exposed container port.

**2. Web Application Configuration**
Web applications typically use nginx as the base image, which already includes a `non-root` user (`nginx`, `UID/GID: 101`).

- Modify the nginx.conf to use non-privileged ports:
  ```nginx
  server {
    listen       8080;  # Use non-privileged port
    server_name  localhost;
    
    # Additional configuration...
  }
  ```
- Dockerfile Adjustments
  ```dockerfile
  # Change directory ownership
    RUN chown -R nginx:nginx /usr/share/nginx/html

  # Run as nginx user
    USER nginx
  ```

**3. Vendor Images with Built-in Non-Root Users**
- Some vendor images already contain non-root users, allowing direct security context configuration in Kubernetes manifests.
  - **Example**: Bitnami images include a non-root user with `UID` `1001`.
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: doc-service
  spec:
    template:
      spec:
        securityContext:
          runAsUser: 1001
          runAsGroup: 1001
          fsGroup: 1001
  ```

**4. Vendor Images Without Non-Root Users**
- For vendor images that do not include a non-root user, additional customization is required:
  1. Fork the original Dockerfile and rebuild the image in your self-hosted registry.
  2. Add user creation and permission modifications:
  ```dockerfile
  # Create non-root user and group
    RUN groupadd -g 1000 appgroup && \
    useradd -u 1000 -g appgroup -s /bin/bash -m appuser
    
  # Change ownership of application directories
    RUN chown -R appuser:appgroup /app

  # Run as non-root user
    USER appuser
  ```

  3. For stateful applications requiring persistent volumes, use initContainers to properly set permissions:
  ```yaml
  initContainers:
  - name: volume-permissions
    image: busybox
    command: ["sh", "-c", "chown -R 1000:1000 /data"]
    volumeMounts:
    - name: data-volume
      mountPath: /data
    securityContext:
      runAsUser: 0  # Temporarily run as root for permission setup

### Verification and Troubleshooting
- Verify User Context
  - To verify the user context of a running container:
  ```bash
  kubectl exec -it [pod-name] -n [namespace] -- id
  ```
 
- Check File Ownership
  - List users defined in the container:
  ```bash
  kubectl exec -it [pod-name] -n [namespace] -- cat /etc/passwd
  ```

### Monitor Compliance
Monitor non-root compliance using Microsoft Defender for Cloud:

1. Navigate to the Azure Portal
2. Access "Microsoft Defender for Cloud"
3. Sort by "severity"
4. Review alerts for "Running containers as root user should be avoided"
5. Click "Take action" to identify affected namespaces

### Conclusion
Implementing non-root security contexts for Kubernetes pods is a critical security practice that requires tailored approaches based on application requirements and base images. When encountering permission issues, analyze container logs for "permission denied" errors and verify the effective permissions inside containers. Consistently follow the principle of least privilege while ensuring application functionality.