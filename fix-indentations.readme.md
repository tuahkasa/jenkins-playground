```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: insights
  labels:
    app: insights
    tier: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: insights
      tier: web
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: insights
        app-group: insights
        tier: web
    spec:
      volumes:
        - name: db-secrets-volume
          secret:
            secretName: portal-db-secret
      containers:
        - name: insights
          image: insights:0.1
          imagePullPolicy: Never
          volumeMounts:
            - name: db-secrets-volume
              mountPath: /db-secrets
          env:
            - name: SECRETS_DIRS
              value: "/db-secrets"
            - name: ALLOWED_HOSTS
              value: "datajar-mobi-stage-test.jamf.build,datajar-sbox-euw2.jamfnebula.com,datajar-dev-euw2.jamfnebula.com,datajar-stage-euw2.jamfnebula.com,datajar-prod-euw2.jamfnebula.com,datajar-sbox-use1.jamfnebula.com,datajar-dev-use1.jamfnebula.com,datajar-stage-use1.jamfnebula.com,datajar-prod-use1.jamfnebula.com,eu.insights.jamf.com,us.insights.jamf.com"
            - name: POD_HOSTNAME
              value: "insights-dev"
            - name: S3_URL
              value: "https://content.datajar.mobi/insights/media"
            - name: SESSION_TIMER
              value: "7200"
            - name: REGION
              value: "EU"
            - name: PORTAL_URL
              value: "http://127.0.0.1:31234"
            - name: MYSQL_HOST
              value: "mysql-insights.default.svc.cluster.local"
            - name: MYSQL_PORT
              value: "3306"
            - name: MYSQL_NAME
              value: "insights"
          livenessProbe:
            tcpSocket:
              port: 5000
            initialDelaySeconds: 600
            periodSeconds: 10
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 5
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
status: {}
```
