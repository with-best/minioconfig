#!/bin/bash
set -e 

CURRENT_HOSTNAME="minio-vm-01" 
MINIO_VERSION="latest"
MINIO_USER="minio-user"
MINIO_PASS='1q2w3e4r!'

echo "=== 1. Hostname & Hosts 설정 ==="
hostnamectl set-hostname $CURRENT_HOSTNAME

# /etc/hosts 중복 방지 로직 추가
if ! grep -q "minio-vm-01" /etc/hosts; then
cat <<EOF >> /etc/hosts
10.40.153.61 minio-vm-01.metanetx.com minio-vm-01
10.40.153.62 minio-vm-02.metanetx.com minio-vm-02
10.40.153.63 minio-vm-03.metanetx.com minio-vm-03
10.40.153.64 minio-vm-04.metanetx.com minio-vm-04
10.40.153.65 minio.metanetx.com       minio-lb
EOF
fi

echo "=== 2. File System & Mount ==="

mkfs.xfs -f -L MINIO_DATA01 /dev/sdb
mkfs.xfs -f -L MINIO_DATA02 /dev/sdc
mkfs.xfs -f -L MINIO_DATA03 /dev/sdd
mkfs.xfs -f -L MINIO_DATA04 /dev/sde

# 마운트 포인트 생성
mkdir -p /mnt/minio_data{01..04}

# fstab 등록 (중복 방지)
for i in {01..04}; do
  if ! grep -q "/mnt/minio_data${i}" /etc/fstab; then
      echo "LABEL=MINIO_DATA${i} /mnt/minio_data${i} xfs defaults,noatime 0 2" | tee -a /etc/fstab
  fi
done

mount -a  

echo "=== 3. MinIO 다운로드 및 사용자 생성 ==="
curl -L https://dl.min.io/server/minio/release/linux-amd64/minio -o /usr/local/bin/minio
chmod +x /usr/local/bin/minio

# 사용자 존재 여부 확인 후 생성
id -u $MINIO_USER &>/dev/null || useradd -r $MINIO_USER -s /sbin/nologin


for i in {01..04}; do
    chown -R $MINIO_USER:$MINIO_USER /mnt/minio_data${i}
done


chown -R $MINIO_USER:$MINIO_USER /etc/pki/minio

echo "=== 5. MinIO 설정 파일 생성 ==="
cat <<EOF > /etc/default/minio
MINIO_ROOT_USER=lokiadmin
MINIO_ROOT_PASSWORD='$MINIO_PASS'
MINIO_VOLUMES="https://minio-vm-{01...04}.metanetx.com:9000/mnt/minio_data{01...04}"
MINIO_OPTS="--address :9000 --console-address :9001 --certs-dir /etc/pki/minio"
EOF

echo "=== 6. Systemd 서비스 등록 ==="
cat <<EOF > /etc/systemd/system/minio.service
[Unit]
Description=MinIO Distributed Cluster
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local/
User=$MINIO_USER
Group=$MINIO_USER
ProtectProc=invisible
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server \$MINIO_OPTS \$MINIO_VOLUMES
Restart=always
LimitNOFILE=65536
TasksMax=infinity
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now minio

echo "Waiting for MinIO to start..."
sleep 5
systemctl status minio --no-pager

echo "=== 7. mc 클라이언트 설정 ==="
curl https://dl.min.io/client/mc/release/linux-amd64/mc --create-dirs -o /usr/local/bin/mc 
chmod +x /usr/local/bin/mc

# --insecure 추가
mc alias set myminio https://minio.metanetx.com:9000 lokiadmin '$MINIO_PASS' --insecure
mc admin info myminio --insecure

echo "Done."
