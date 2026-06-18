# Runbook: Unsigned Image Rejected

## Trigger
Pod không start được, admission webhook reject với lỗi:
```
admission webhook "policy.sigstore.dev" denied the request:
image ghcr.io/andoan2210/w10-api:<tag> is not signed
```

## Root Cause
Image chưa được ký bằng Cosign private key, hoặc chữ ký không match public key trong ClusterImagePolicy.

## Steps

### 1. Verify image có signature chưa
```bash
cosign verify \
  --key signing/cosign.pub \
  ghcr.io/andoan2210/w10-api:<tag>
```

### 2. Nếu image chưa ký — ký thủ công (emergency only)
```bash
# Chỉ dùng khi CI bị lỗi, cần deploy gấp
cosign sign --key cosign.key \
  ghcr.io/andoan2210/w10-api:<tag>
```

### 3. Nếu image đã ký nhưng vẫn bị reject — kiểm tra public key
```bash
# Xem ClusterImagePolicy đang dùng key nào
kubectl get clusterimagepolicy require-cosign-signature -o yaml

# So sánh với signing/cosign.pub trong repo
cat signing/cosign.pub
```

### 4. Nếu key mismatch — rotate key
1. Generate key pair mới: `cosign generate-key-pair`
2. Update GitHub Secret `COSIGN_PRIVATE_KEY`
3. Commit `signing/cosign.pub` mới vào repo
4. Re-build image để ký lại với key mới

### 5. Kiểm tra namespace label
```bash
# Policy Controller chỉ enforce namespace có label này
kubectl get namespace demo --show-labels
# Phải có: policy.sigstore.dev/include=true
```

## Prevention
- Không bao giờ deploy image ngoài CI pipeline
- Mọi image production phải đi qua build-push.yml
