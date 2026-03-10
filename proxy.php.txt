<?php
/**
 * StreamZone Proxy — InfinityFree Compatible
 * index.html এর পাশে রাখুন
 */

// Block direct browser access (optional security)
error_reporting(0);
ini_set('display_errors', 0);

// CORS headers
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, OPTIONS');
header('Access-Control-Allow-Headers: *');
header('X-Robots-Tag: noindex, nofollow');

if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    http_response_code(204); exit;
}

// ── Credentials (hidden from user) ──
define('SZ_HOST', 'kstv.us');
define('SZ_PORT', '8080');
define('SZ_USER', 'vf4Q6h41Am');
define('SZ_PASS', '1376279062');
define('SZ_BASE', 'http://' . SZ_HOST . ':' . SZ_PORT);

$type = isset($_GET['type']) ? $_GET['type'] : '';

// ════════════════════════════════
// API: get_live_categories / get_live_streams
// ════════════════════════════════
if ($type === 'api') {
    $allowed = ['get_live_categories', 'get_live_streams', 'get_vod_categories', 'get_live_stream_info'];
    $action  = isset($_GET['action']) ? $_GET['action'] : '';
    if (!in_array($action, $allowed)) {
        http_response_code(400);
        echo json_encode(['error' => 'Invalid action']);
        exit;
    }

    $url = SZ_BASE . '/player_api.php'
         . '?username=' . SZ_USER
         . '&password=' . SZ_PASS
         . '&action='   . urlencode($action);

    $data = sz_fetch($url);
    header('Content-Type: application/json; charset=utf-8');
    header('Cache-Control: public, max-age=120'); // 2 min cache
    echo $data;
    exit;
}

// ════════════════════════════════
// STREAM: HLS m3u8 playlist
// ════════════════════════════════
if ($type === 'stream') {
    $id = isset($_GET['id']) ? intval($_GET['id']) : 0;
    if (!$id) { http_response_code(400); echo 'Missing id'; exit; }

    $url     = SZ_BASE . '/live/' . SZ_USER . '/' . SZ_PASS . '/' . $id . '.m3u8';
    $content = sz_fetch($url);

    if (!$content || strpos($content, '#EXTM3U') === false) {
        // Maybe it's a redirect or different format — try ts directly
        header('Content-Type: application/vnd.apple.mpegurl');
        header('Cache-Control: no-cache, no-store');
        echo $content ?: '#EXTM3U' . "\n" . '#EXT-X-ENDLIST';
        exit;
    }

    // Rewrite segment URLs through this proxy
    $content = rewrite_m3u8($content, $id);
    header('Content-Type: application/vnd.apple.mpegurl');
    header('Cache-Control: no-cache, no-store');
    echo $content;
    exit;
}

// ════════════════════════════════
// SEGMENT: .ts video chunks
// ════════════════════════════════
if ($type === 'ts') {
    $rawUrl  = isset($_GET['u']) ? $_GET['u'] : '';
    $relPath = isset($_GET['p']) ? $_GET['p'] : '';

    if ($rawUrl) {
        $segUrl = base64_decode($rawUrl); // URL is base64 encoded for safety
    } elseif ($relPath) {
        $segUrl = SZ_BASE . '/' . ltrim($relPath, '/');
    } else {
        http_response_code(400); echo 'Missing segment'; exit;
    }

    // Validate it points to our server only
    if (strpos($segUrl, SZ_HOST) === false) {
        http_response_code(403); echo 'Forbidden'; exit;
    }

    header('Content-Type: video/MP2T');
    header('Cache-Control: no-cache, no-store');
    header('Access-Control-Allow-Origin: *');

    // Stream directly to output (memory efficient)
    $ch = curl_init($segUrl);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => false,
        CURLOPT_WRITEFUNCTION  => function($ch, $data) {
            echo $data;
            return strlen($data);
        },
        CURLOPT_FOLLOWLOCATION => true,
        CURLOPT_TIMEOUT        => 30,
        CURLOPT_CONNECTTIMEOUT => 10,
        CURLOPT_USERAGENT      => 'Mozilla/5.0 (Linux; Android 10) AppleWebKit/537.36',
        CURLOPT_SSL_VERIFYPEER => false,
        CURLOPT_HTTPHEADER     => [
            'Accept: */*',
            'Connection: keep-alive',
        ],
    ]);
    curl_exec($ch);
    curl_close($ch);
    exit;
}

// ════════════════════════════════
// LOGO IMAGE PROXY (optional — prevents mixed content)
// ════════════════════════════════
if ($type === 'img') {
    $src = isset($_GET['src']) ? base64_decode($_GET['src']) : '';
    if (!$src || !filter_var($src, FILTER_VALIDATE_URL)) {
        http_response_code(400); exit;
    }
    $img = sz_fetch($src, false, true);
    $ct  = 'image/png';
    if (str_contains($src, '.jpg') || str_contains($src, '.jpeg')) $ct = 'image/jpeg';
    if (str_contains($src, '.webp')) $ct = 'image/webp';
    header('Content-Type: ' . $ct);
    header('Cache-Control: public, max-age=86400');
    echo $img;
    exit;
}

http_response_code(400);
echo json_encode(['error' => 'Unknown request type']);
exit;

// ════════════════════════════════
// HELPERS
// ════════════════════════════════

function sz_fetch($url, $stream = false, $binary = false) {
    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_FOLLOWLOCATION => true,
        CURLOPT_MAXREDIRS      => 5,
        CURLOPT_TIMEOUT        => 20,
        CURLOPT_CONNECTTIMEOUT => 10,
        CURLOPT_USERAGENT      => 'Mozilla/5.0 (Linux; Android 10) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120 Mobile Safari/537.36',
        CURLOPT_SSL_VERIFYPEER => false,
        CURLOPT_SSL_VERIFYHOST => false,
        CURLOPT_HTTPHEADER     => [
            'Accept: */*',
            'Accept-Language: en-US,en;q=0.9',
            'Connection: keep-alive',
            'Referer: http://' . SZ_HOST . ':' . SZ_PORT . '/',
        ],
    ]);
    $res  = curl_exec($ch);
    $err  = curl_errno($ch);
    curl_close($ch);
    if ($err) return '';
    return $res ?: '';
}

function rewrite_m3u8($content, $streamId) {
    // Build proxy base URL dynamically
    $scheme = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? 'https' : 'http';
    $host   = $_SERVER['HTTP_HOST'];
    $dir    = rtrim(dirname($_SERVER['REQUEST_URI']), '/');
    $base   = $scheme . '://' . $host . $dir . '/proxy.php';

    $lines  = explode("\n", $content);
    $out    = [];

    foreach ($lines as $line) {
        $line = rtrim($line);
        if ($line === '' || $line[0] === '#') {
            $out[] = $line;
            continue;
        }

        // It's a segment URL
        if (filter_var($line, FILTER_VALIDATE_URL)) {
            // Absolute URL
            if (substr($line, -5) === '.m3u8') {
                $out[] = $base . '?type=stream&id=' . $streamId;
            } else {
                $out[] = $base . '?type=ts&u=' . base64_encode($line);
            }
        } else {
            // Relative path
            if (substr($line, -5) === '.m3u8') {
                $out[] = $base . '?type=stream&id=' . $streamId;
            } else {
                $out[] = $base . '?type=ts&p=' . urlencode($line);
            }
        }
    }

    return implode("\n", $out);
}
