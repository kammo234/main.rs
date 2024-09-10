use hyper::{Body, Request, Response, Server};
use hyper::service::{make_service_fn, service_fn};
use std::fs::File;
use std::io::Read;
use std::path::Path;
use tokio::fs::File as TokioFile;
use tokio::io::AsyncReadExt;

async fn handle_request(req: Request<Body>) -> Result<Response<Body>, hyper::Error> {
    let path = match req.uri().path() {
        "/" => "index.html",
        p => &p[1..], // Remove leading "/"
    };

    let file_path = Path::new(path);
    if file_path.exists() && file_path.is_file() {
        let mut file = TokioFile::open(file_path).await.unwrap();
        let mut contents = Vec::new();
        file.read_to_end(&mut contents).await.unwrap();
        Ok(Response::new(Body::from(contents)))
    } else {
        Ok(Response::builder()
            .status(404)
            .body(Body::from("File not found"))
            .unwrap())
    }
}

#[tokio::main]
async fn main() {
    let make_svc = make_service_fn(|_conn| async { Ok::<_, hyper::Error>(service_fn(handle_request)) });

    let addr = ([127, 0, 0, 1], 8080).into();
    let server = Server::bind(&addr).serve(make_svc);

    println!("Listening on http://{}", addr);

    if let Err(e) = server.await {
        eprintln!("Server error: {}", e);
    }
}
