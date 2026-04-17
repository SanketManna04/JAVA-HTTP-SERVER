import java.io.*;
import java.net.*;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.concurrent.*;

/**
 * MiniHttpServer - A multithreaded HTTP server that handles basic GET requests.
 * Uses a thread pool to handle concurrent connections efficiently.
 */
public class MiniHttpServer {

    private static final int DEFAULT_PORT = 8080;
    private static final int THREAD_POOL_SIZE = 10;

    private final int port;
    private final ExecutorService threadPool;
    private volatile boolean running = false;
    private ServerSocket serverSocket;

    public MiniHttpServer(int port) {
        this.port = port;
        this.threadPool = Executors.newFixedThreadPool(THREAD_POOL_SIZE);
    }

    public void start() throws IOException {
        serverSocket = new ServerSocket(port);
        running = true;
        log("Server started on http://localhost:" + port);
        log("Thread pool size: " + THREAD_POOL_SIZE);
        log("Waiting for connections...\n");

        // Register shutdown hook for graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(this::stop));

        while (running) {
            try {
                Socket clientSocket = serverSocket.accept();
                // Hand off to thread pool — non-blocking for the main thread
                threadPool.submit(new RequestHandler(clientSocket));
            } catch (SocketException e) {
                if (running) {
                    log("ERROR: " + e.getMessage());
                }
            }
        }
    }

    public void stop() {
        running = false;
        try {
            if (serverSocket != null && !serverSocket.isClosed()) {
                serverSocket.close();
            }
        } catch (IOException e) {
            log("Error closing server socket: " + e.getMessage());
        }
        threadPool.shutdown();
        log("Server stopped.");
    }

    private static void log(String message) {
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("HH:mm:ss"));
        System.out.println("[" + timestamp + "] " + message);
    }

    // -----------------------------------------------------------------------
    // Inner class: handles one HTTP request on its own thread
    // -----------------------------------------------------------------------
    static class RequestHandler implements Runnable {

        private final Socket clientSocket;

        RequestHandler(Socket clientSocket) {
            this.clientSocket = clientSocket;
        }

        @Override
        public void run() {
            String clientAddr = clientSocket.getInetAddress().getHostAddress();
            log("Thread " + Thread.currentThread().getName() + " handling request from " + clientAddr);

            try (
                BufferedReader in  = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                PrintWriter    out = new PrintWriter(clientSocket.getOutputStream(), true)
            ) {
                // --- Parse request line ---
                String requestLine = in.readLine();
                if (requestLine == null || requestLine.isEmpty()) {
                    return;
                }
                log("Request: " + requestLine);

                // Read and discard headers (needed to drain the socket)
                String headerLine;
                while ((headerLine = in.readLine()) != null && !headerLine.isEmpty()) {
                    // headers read but not used for this minimal server
                }

                String[] parts = requestLine.split(" ");
                if (parts.length < 2) {
                    sendResponse(out, 400, "Bad Request", "text/plain", "400 Bad Request");
                    return;
                }

                String method = parts[0];
                String path   = parts[1];

                if (!method.equals("GET")) {
                    sendResponse(out, 405, "Method Not Allowed", "text/plain",
                            "405 Method Not Allowed: Only GET is supported.");
                    return;
                }

                // --- Route the GET request ---
                handleGet(out, path);

            } catch (IOException e) {
                log("ERROR handling request: " + e.getMessage());
            } finally {
                try { clientSocket.close(); } catch (IOException ignored) {}
            }
        }

        private void handleGet(PrintWriter out, String path) {
            switch (path) {
                case "/":
                    sendResponse(out, 200, "OK", "text/html", buildHomePage());
                    break;

                case "/hello":
                    sendResponse(out, 200, "OK", "text/plain",
                            "Hello, World! Served by thread: " + Thread.currentThread().getName());
                    break;

                case "/info":
                    sendResponse(out, 200, "OK", "text/html", buildInfoPage());
                    break;

                case "/time":
                    String time = LocalDateTime.now()
                            .format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
                    sendResponse(out, 200, "OK", "text/plain", "Current server time: " + time);
                    break;

                default:
                    sendResponse(out, 404, "Not Found", "text/html", buildNotFoundPage(path));
                    break;
            }
        }

        // --- HTTP response builder ---
        private void sendResponse(PrintWriter out, int statusCode, String statusText,
                                  String contentType, String body) {
            byte[] bodyBytes = body.getBytes();
            out.print("HTTP/1.1 " + statusCode + " " + statusText + "\r\n");
            out.print("Content-Type: " + contentType + "\r\n");
            out.print("Content-Length: " + bodyBytes.length + "\r\n");
            out.print("Connection: close\r\n");
            out.print("\r\n");
            out.print(body);
            out.flush();
            log("Response: " + statusCode + " " + statusText + " -> " + body.length() + " bytes");
        }

        // --- HTML page builders ---
        private String buildHomePage() {
            return "<!DOCTYPE html><html><head><title>Mini HTTP Server</title>"
                 + "<style>body{font-family:monospace;background:#1e1e2e;color:#cdd6f4;"
                 + "max-width:600px;margin:60px auto;padding:20px}"
                 + "h1{color:#cba6f7}a{color:#89b4fa;display:block;margin:8px 0}"
                 + "code{background:#313244;padding:2px 6px;border-radius:4px}</style></head>"
                 + "<body><h1>🚀 Mini HTTP Server</h1>"
                 + "<p>Multithreaded Java server is running!</p>"
                 + "<p>Available routes:</p>"
                 + "<a href='/hello'><code>GET /hello</code> — Hello World</a>"
                 + "<a href='/time'><code>GET /time</code> — Current server time</a>"
                 + "<a href='/info'><code>GET /info</code> — Server info</a>"
                 + "</body></html>";
        }

        private String buildInfoPage() {
            Runtime rt = Runtime.getRuntime();
            long usedMb  = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
            long totalMb = rt.totalMemory() / (1024 * 1024);
            return "<!DOCTYPE html><html><head><title>Server Info</title>"
                 + "<style>body{font-family:monospace;background:#1e1e2e;color:#cdd6f4;"
                 + "max-width:600px;margin:60px auto;padding:20px}"
                 + "h1{color:#cba6f7}table{width:100%;border-collapse:collapse}"
                 + "td{padding:8px;border-bottom:1px solid #313244}"
                 + "td:first-child{color:#a6e3a1}</style></head>"
                 + "<body><h1>⚙️ Server Info</h1><table>"
                 + "<tr><td>Java version</td><td>" + System.getProperty("java.version") + "</td></tr>"
                 + "<tr><td>OS</td><td>" + System.getProperty("os.name") + "</td></tr>"
                 + "<tr><td>Thread</td><td>" + Thread.currentThread().getName() + "</td></tr>"
                 + "<tr><td>Available CPUs</td><td>" + rt.availableProcessors() + "</td></tr>"
                 + "<tr><td>Memory used</td><td>" + usedMb + " MB / " + totalMb + " MB</td></tr>"
                 + "<tr><td>Time</td><td>" + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")) + "</td></tr>"
                 + "</table></body></html>";
        }

        private String buildNotFoundPage(String path) {
            return "<!DOCTYPE html><html><head><title>404 Not Found</title>"
                 + "<style>body{font-family:monospace;background:#1e1e2e;color:#cdd6f4;"
                 + "text-align:center;padding:80px}h1{color:#f38ba8;font-size:4em}"
                 + "a{color:#89b4fa}</style></head>"
                 + "<body><h1>404</h1><p>Path <b>" + path + "</b> not found.</p>"
                 + "<a href='/'>← Back to home</a></body></html>";
        }
    }

    // -----------------------------------------------------------------------
    public static void main(String[] args) throws IOException {
        int port = args.length > 0 ? Integer.parseInt(args[0]) : DEFAULT_PORT;
        new MiniHttpServer(port).start();
    }
}
