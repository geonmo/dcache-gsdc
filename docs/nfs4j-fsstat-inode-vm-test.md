# nfs4j `getFsStat(Inode)` VM 검증 지시서

다른 머신(테스트 VM)에서 nfs4j 변경을 빌드하고, 커널 NFS 클라이언트로 경로별 `df`가 동작하는지 확인하기 위한 절차입니다.
**CMS T2 운영 시스템과 무관한 VM에서만** 실행합니다. 운영 호스트에서는 실행하지 않습니다.

## 1. 배경

- dCache PR #8276(머지됨)은 NFS door에서 경로별 용량을 보고하지 못합니다. nfs4j의 `VirtualFileSystem.getFsStat()`에 인자가 없기 때문입니다.
- 리눅스 커널(v5.14, master 동일)은 이미 `statfs` 대상 경로 **자신**의 file handle을 보냅니다.
  - `fs/nfs/super.c` `nfs_statfs()` → `NFS_FH(d_inode(dentry))`
  - NFSv4: `PUTFH(fh)` + `GETATTR(SPACE_*, FILES_*)`
  - NFSv3: `FSSTAT(fh)`
  - 클라이언트 쪽 statfs 캐시가 없습니다(호출마다 RPC). NFSv3 `invarsec`는 무시됩니다.
- 그런데 nfs4j가 그 inode를 버립니다. 이번 변경은 inode를 VFS까지 전달합니다.

## 2. 브랜치

- 저장소: `gh repo clone geonmo/nfs4j` (upstream: dCache/nfs4j)
- 브랜치: `fsstat-inode` (기준: upstream master `1ddfd3d6`)

| 커밋 | 내용 | PR 포함 여부 |
|---|---|---|
| `c39878ab` vfs: pass the queried object to getFsStat | 본 변경 + 단위 테스트 | 포함 |
| `6a7e6d98` DO NOT MERGE: basic-server: … | E2E 전용. `LocalFileSystem#getFsStat(Inode)`가 조회 경로의 `FileStore`를 보고하고 `E2E getFsStat` 로그를 남김 | **PR 전에 제거** |

본 변경 요약 (`core/`):
- `VirtualFileSystem`: `default FsStat getFsStat(Inode inode) { return getFsStat(); }` 추가 (하위 호환)
- `OperationGETATTR`: `FATTR4_SPACE_*` / `FILES_*` → `fs.getFsStat(inode)`
- `NfsServerV3` FSSTAT: `arg1.fsroot` → `fs.getFsStat(inode)`
- `ForwardingFileSystem`: 전달 / `PseudoFs`: `innerInode(inode)`로 변환 후 전달
- `VfsCache`: inode별 캐시(`Cache<Opaque, FsStat>`, 수명은 기존 fs-stat lifetime). 기존 `getFsStat()` 캐시는 그대로
- 테스트: `OperationGETATTRTest.shouldReportSpaceOfQueriedObject`, `VfsCacheTest.shouldPassInodeToFsStat` / `shouldCacheFsStatPerInode`, `PseudoFsTest.shouldPassInnerInodeToFsStat`

## 3. VM 준비

요구사항: AlmaLinux 9(또는 RHEL9 계열), root 권한, `/tmp` 여유 **5GB 이상**(BerkeleyDB 테스트가 요구), 로컬 디스크(`/tmp`·작업 디렉토리를 NFS 위에 두지 않기).

```bash
sudo dnf install -y java-21-openjdk-devel git nfs-utils rpcbind tar curl
# Maven 3.9.x (배포판 maven은 오래됨)
curl -fsSL -o /tmp/mvn.tgz https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz
curl -fsSL https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz.sha512 | awk '{print $1"  /tmp/mvn.tgz"}' | sha512sum -c -
sudo mkdir -p /opt/maven && sudo tar -xzf /tmp/mvn.tgz -C /opt/maven --strip-components=1
export PATH=/opt/maven/bin:$PATH
java -version; mvn -v; df -h /tmp

# 커널 nfsd가 떠 있으면 끈다(포트·rpcbind 등록 충돌 방지). rpcbind는 켜 둔다.
sudo systemctl stop nfs-server 2>/dev/null; sudo systemctl start rpcbind
```

## 4. 빌드와 단위 테스트

```bash
gh repo clone geonmo/nfs4j && cd nfs4j
git checkout fsstat-inode
mvn -B install 2>&1 | tee build.log
grep -E 'Tests run:.*Fail|BUILD' build.log | tail -5
```

기대 결과: `BUILD SUCCESS`, core 테스트 실패 0.
참고: 제어 노드에서는 core 459개 중 `BerkeleyDBClientStoreTest`만 디스크 환경 때문에 실패했습니다(`/tmp` 여유 2.3GB < 5GB, NFS 위 임시 디렉토리 삭제 실패). 변경과 무관한 테스트입니다. VM에서는 통과해야 합니다.

변경 관련 테스트만 빠르게:
```bash
mvn -B -pl core -am test -Dtest='OperationGETATTRTest,VfsCacheTest,PseudoFsTest,FsStatTest' -Dsurefire.failIfNoSpecifiedTests=false
```
기대: OperationGETATTRTest 2, VfsCacheTest 21, PseudoFsTest 27, FsStatTest 1 모두 통과.

## 5. E2E: 커널 클라이언트로 경로별 df 확인

### 5.1 export 트리 준비 (크기가 다른 tmpfs를 하위 디렉토리에 마운트)

```bash
sudo mkdir -p /srv/nfs4j/data/small /srv/nfs4j/data/medium /srv/nfs4j/data/plain
sudo mount -t tmpfs -o size=100m tmpfs /srv/nfs4j/data/small
sudo mount -t tmpfs -o size=300m tmpfs /srv/nfs4j/data/medium
sudo dd if=/dev/zero of=/srv/nfs4j/data/small/f bs=1M count=10
echo '/data *(rw,no_root_squash)' | sudo tee /srv/nfs4j.exports
df -B1 --output=size,used,avail,target /srv/nfs4j/data /srv/nfs4j/data/small /srv/nfs4j/data/medium   # 로컬 기준값
```

### 5.2 서버 실행 (포트 2050)

```bash
JAR=$(ls basic-server/target/*jar-with-dependencies.jar)
sudo java -jar $JAR -root /srv/nfs4j -exports /srv/nfs4j.exports -port 2050 -nfsvers 0 > /tmp/nfs4j-server.log 2>&1 &
sleep 5; tail -5 /tmp/nfs4j-server.log
```

### 5.3 마운트

```bash
sudo mkdir -p /mnt/v4 /mnt/v3 /mnt/sub
sudo mount -t nfs4 -o vers=4.1,port=2050 127.0.0.1:/data /mnt/v4
sudo mount -t nfs -o vers=3,proto=tcp,port=2050,mountport=2050,mountproto=tcp,nolock 127.0.0.1:/data /mnt/v3
sudo mount -t nfs4 -o vers=4.1,port=2050 127.0.0.1:/data/small /mnt/sub
```

### 5.4 확인

```bash
DF='df -B1 --output=size,used,avail,target'
for m in /mnt/v4 /mnt/v3; do $DF $m $m/plain $m/small $m/medium $m/small/f; done
$DF /mnt/sub
df -B1 --output=size,target | grep -E '/mnt/(v4|v3|sub)'      # 인자 없는 df: 마운트 루트 기준
grep 'E2E getFsStat' /tmp/nfs4j-server.log | tail -20
```

기대 결과:

| 대상 | 기대 size |
|---|---|
| `/mnt/v4`, `/mnt/v3`, `…/plain` | `/srv/nfs4j/data`가 있는 로컬 FS 크기 |
| `…/small`, `…/small/f`(파일), `/mnt/sub` | 약 100MB (used 약 10MB) |
| `…/medium` | 약 300MB |

- 서버 로그의 `E2E getFsStat <path>`가 df 대상 경로와 일치해야 합니다(파일이면 파일 경로).
- 같은 df를 여러 번 실행하면 실행할 때마다 로그 줄이 늘어나야 합니다(클라이언트 캐시 없음).

### 5.5 (선택) 기준선 비교

변경 전 동작을 보려면 서버를 멈추고 upstream master로 빌드해서 같은 df를 실행합니다. 모든 경로가 루트 FS 크기로 같아야 합니다.
```bash
git checkout 1ddfd3d6 && mvn -B -q install -DskipTests   # 서버 재시작 전 마운트 해제
```

### 5.6 정리

```bash
sudo umount /mnt/sub /mnt/v4 /mnt/v3
sudo pkill -f 'jar-with-dependencies.jar -root /srv/nfs4j'
sudo umount /srv/nfs4j/data/small /srv/nfs4j/data/medium
mount | grep -E 'nfs4j|/mnt/(v4|v3|sub)' || echo clean
```

## 6. 주의사항

- `LocalFileSystem`은 inode를 메모리에서 무작위로 만듭니다. 서버를 재시작하면 기존 마운트가 stale이 되므로 다시 마운트합니다.
- 포트 2049(커널 nfsd)와 겹치지 않게 2050을 씁니다. 방화벽이 켜져 있어도 127.0.0.1이므로 영향이 없습니다.
- 클라이언트 IP는 VFS로 전달되지 않습니다(이번 변경 범위 밖).

## 7. 보고할 것

1. `mvn install` 결과(테스트 수, 실패 목록)
2. 5.4의 df 출력과 서버 로그 `E2E getFsStat` 줄
3. (선택) 5.5 기준선 출력
4. 예상과 다른 점

## 8. 이후 단계 (승인 후)

1. dCache/nfs4j에 이슈를 먼저 올려 방향을 확인합니다. 근거로 커널 동작, dCache #8276, nfs4j #133(BNL 사용자별 space token 사례)을 듭니다.
2. `DO NOT MERGE` 커밋을 뺀 PR용 브랜치(`git branch fsstat-inode-pr c39878ab`)로 커밋 1개짜리 PR을 냅니다. VM에서 본 변경을 고쳤다면 그 커밋에 squash한 뒤 만듭니다. Signed-off-by는 포함하고 세션 링크는 넣지 않습니다.
3. nfs4j 릴리스 뒤 dCache에서 할 일:
   - `ChimeraVfs`에서 `getFsStat(Inode)` 구현
   - reservation/link group 키 캐시 추가
   - `nfs.capacity.reservation` 필수 조건 제거
