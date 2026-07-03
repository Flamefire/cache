Bootstrap: docker
From: ubuntu:16.04

%labels
    Author opencode
    Description Ubuntu 16.04 Node 24 Azure SDK SIGSEGV reproduction

%environment
    export DEBIAN_FRONTEND=noninteractive

%post
    export DEBIAN_FRONTEND=noninteractive

    # Update package lists and install required tools
    apt-get update && apt-get install -y curl xz-utils ca-certificates zstd

    # Install Node 24 (glibc-2.17 build) to /usr/local — this is the default node
    # Source: https://archives.boost.io/misc/node/node-v24.15.0-linux-x64-glibc-217.tar.xz
    curl -sL https://archives.boost.io/misc/node/node-v24.15.0-linux-x64-glibc-217.tar.xz | tar -xJ --strip-components 1 -C /usr/local

    # Install Node 20 (glibc-2.17 build) to /opt/node20 for comparison
    # Source: https://archives.boost.io/misc/node/node-v20.9.0-linux-x64-glibc-217.tar.xz
    mkdir -p /opt/node20
    curl -sL https://archives.boost.io/misc/node/node-v20.9.0-linux-x64-glibc-217.tar.xz | tar -xJ --strip-components 1 -C /opt/node20

    # Print system info for diagnostics
    echo "=== System Info ==="
    uname -r
    ldd --version
    node --version
    node -p "JSON.stringify(process.versions)"
    echo "Node 20:"
    /opt/node20/bin/node --version

    # Create test directory
    mkdir -p /opt/cache-test

    # Install Azure Storage Blob SDK (the SDK whose pipeline triggers the crash)
    cd /opt/cache-test && npm install @azure/storage-blob

    # ------------------------------------------------------------------
    # test-azsdk.mjs — exercises BlockBlobClient.getProperties() through
    # the full Azure SDK pipeline (NodeHttpClient.makeRequest → https.request).
    # This is the code path that SIGSEGVs in the full GHA action context.
    # Exit code 139 = SIGSEGV crash; exit 0 = no crash (normal JS error or success).
    # ------------------------------------------------------------------
    cat > /opt/cache-test/test-azsdk.mjs <<'AZSDK_EOF'
import { BlockBlobClient } from '@azure/storage-blob';
console.error('[TEST] imported, creating BlockBlobClient...');
const client = new BlockBlobClient(
  'https://productionresultssa0.blob.core.windows.net/actions-cache/test'
);
console.error('[TEST] calling getProperties...');
try {
  const props = await client.getProperties();
  console.error('[TEST] getProperties succeeded:', props.contentLength);
} catch (e) {
  console.error('[TEST] getProperties threw:', e.message);
}
process.exit(0);
AZSDK_EOF

    # ------------------------------------------------------------------
    # test-https.mjs — bare https.request() to the same Azure endpoint.
    # This does NOT crash on its own (only crashes through the Azure SDK
    # pipeline in the full GHA action context).
    # Exit code 139 = SIGSEGV crash; exit 0 = no crash.
    # ------------------------------------------------------------------
    cat > /opt/cache-test/test-https.mjs <<'HTTPS_EOF'
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const https = require('node:https');
const agent = new https.Agent({ keepAlive: true });
const options = {
  agent,
  hostname: 'productionresultssa0.blob.core.windows.net',
  path: '/actions-cache/test',
  port: '',
  method: 'HEAD',
  headers: { 'x-ms-version': '2025-11-05', 'User-Agent': 'azsdk-js-azure-storage-blob/12.32.0' },
};
console.log('before https.request');
const req = https.request(options, (res) => {
  console.log('response:', res.statusCode);
  res.resume();
  process.exit(0);
});
console.log('after https.request');
req.on('socket', (s) => {
  console.log('socket assigned');
  s.on('lookup', (e,a) => console.log('DNS:', a));
  s.on('connect', () => console.log('TCP connected'));
  s.on('secureConnect', () => console.log('TLS done'));
});
req.on('error', (e) => { console.log('error:', e.message); process.exit(1); });
req.end();
console.log('after req.end()');
HTTPS_EOF

%test
    # Run the Azure SDK test with Node 24 (default).
    # Exit code 139 = SIGSEGV crash (the bug we are reproducing); exit 0 = no crash.
    node /opt/cache-test/test-azsdk.mjs

%runscript
    # Comprehensive SIGSEGV reproduction test runner.
    # Runs both test scripts with Node 24 and Node 20, reporting exit codes.
    # Exit code 139 = SIGSEGV (crash); exit code 0 = no crash.

    echo "========================================"
    echo "  Ubuntu 16.04 Node 24 Azure SDK Test"
    echo "========================================"
    echo ""
    echo "=== System Info ==="
    uname -r
    ldd --version 2>&1 | head -1
    echo "Node 24: $(node --version)"
    echo "Node 20: $(/opt/node20/bin/node --version)"
    echo ""

    # Test 1: bare https.request with Node 24
    # Should NOT crash (crash only occurs through the Azure SDK pipeline).
    echo "=== Test 1: https.request (Node 24) ==="
    node /opt/cache-test/test-https.mjs
    echo "Exit code: $?"
    echo ""

    # Test 2: bare https.request with Node 20
    # Baseline comparison — should NOT crash.
    echo "=== Test 2: https.request (Node 20) ==="
    /opt/node20/bin/node /opt/cache-test/test-https.mjs
    echo "Exit code: $?"
    echo ""

    # Test 3: Azure SDK getProperties with Node 24
    # PRIMARY CRASH REPRODUCTION PATH — exit code 139 indicates SIGSEGV.
    echo "=== Test 3: Azure SDK getProperties (Node 24) ==="
    node /opt/cache-test/test-azsdk.mjs
    echo "Exit code: $?"
    echo ""

    # Test 4: Azure SDK getProperties with Node 20
    # Baseline comparison — Node 20 does not exhibit the crash.
    echo "=== Test 4: Azure SDK getProperties (Node 20) ==="
    /opt/node20/bin/node /opt/cache-test/test-azsdk.mjs
    echo "Exit code: $?"
    echo ""

    echo "========================================"
    echo "  Exit code 139 = SIGSEGV (crash)"
    echo "  Exit code 0   = no crash"
    echo "========================================"
