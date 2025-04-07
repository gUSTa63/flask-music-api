from flask import Flask, request, jsonify, send_file
import yt_dlp
import os
import uuid

app = Flask(__name__)

DOWNLOAD_FOLDER = "downloads"
os.makedirs(DOWNLOAD_FOLDER, exist_ok=True)

@app.route("/download", methods=["POST"])
def download_music():
    data = request.get_json()
    url = data.get("url")

    if not url:
        return jsonify({"error": "URL não fornecida"}), 400

    unique_id = str(uuid.uuid4())
    output_path = os.path.join(DOWNLOAD_FOLDER, f"{unique_id}.%(ext)s")

    ydl_opts = {
        'format': 'bestaudio/best',
        'outtmpl': output_path,
        'postprocessors': [{
            'key': 'FFmpegExtractAudio',
            'preferredcodec': 'mp3',
            'preferredquality': '192',
        }],
        'quiet': True
    }

    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            filename = ydl.prepare_filename(info)
            filename = filename.replace(".webm", ".mp3").replace(".m4a", ".mp3")
    except Exception as e:
        return jsonify({"error": str(e)}), 500

    return jsonify({
        "filename": os.path.basename(filename),
        "url": f"/musica/{os.path.basename(filename)}"
    })

@app.route("/musica/<nome>")
def serve_file(nome):
    caminho = os.path.join(DOWNLOAD_FOLDER, nome)
    if os.path.exists(caminho):
        return send_file(caminho, as_attachment=True)
    return jsonify({"error": "Arquivo não encontrado"}), 404

if __name__ == "__main__":
    app.run(debug=True)
