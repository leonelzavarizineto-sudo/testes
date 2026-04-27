import argparse
import csv
import os
import re
import sys
import zipfile
from dataclasses import dataclass
from pathlib import Path
from typing import Iterable


PADROES_ERRO = {
    "CRITICO": [
        r"\btrip\b",
        r"\bfatal\b",
        r"\bcritical\b",
        r"\bmajor\b",
        r"\bemergency\b",
        r"\bshutdown\b",
    ],
    "ERRO": [
        r"\berror\b",
        r"\bfail(?:ed|ure)?\b",
        r"\bfault\b",
        r"\bexception\b",
        r"\balarm\b",
        r"\btimeout\b",
    ],
    "AVISO": [
        r"\bwarning\b",
        r"\bwarn\b",
        r"\bminor\b",
        r"\blost\b",
        r"\bunavailable\b",
    ],
}


EXTENSOES_LOG = {".log", ".txt", ".csv"}
ENCODINGS = ("utf-8-sig", "utf-8", "latin-1", "cp1252")


@dataclass
class Evento:
    arquivo: str
    linha: int
    severidade: str
    termo: str
    texto: str


def compilar_padroes() -> list[tuple[str, re.Pattern]]:
    padroes = []
    for severidade, expressoes in PADROES_ERRO.items():
        for expressao in expressoes:
            padroes.append((severidade, re.compile(expressao, re.IGNORECASE)))
    return padroes


def decodificar_bytes(dados: bytes) -> str:
    for encoding in ENCODINGS:
        try:
            return dados.decode(encoding)
        except UnicodeDecodeError:
            continue
    return dados.decode("utf-8", errors="replace")


def deve_analisar(nome_arquivo: str) -> bool:
    return Path(nome_arquivo).suffix.lower() in EXTENSOES_LOG


def ler_arquivos(caminho: Path) -> Iterable[tuple[str, str]]:
    if caminho.is_file() and caminho.suffix.lower() == ".zip":
        with zipfile.ZipFile(caminho, "r") as zip_file:
            for info in zip_file.infolist():
                if info.is_dir() or not deve_analisar(info.filename):
                    continue
                with zip_file.open(info) as arquivo:
                    yield info.filename, decodificar_bytes(arquivo.read())
        return

    if caminho.is_file():
        yield caminho.name, decodificar_bytes(caminho.read_bytes())
        return

    if caminho.is_dir():
        for arquivo in caminho.rglob("*"):
            if arquivo.is_file() and (arquivo.suffix.lower() == ".zip" or deve_analisar(arquivo.name)):
                yield from ler_arquivos(arquivo)
        return

    raise FileNotFoundError(f"Caminho nao encontrado: {caminho}")


def analisar_conteudo(nome_arquivo: str, conteudo: str, padroes: list[tuple[str, re.Pattern]]) -> list[Evento]:
    eventos = []
    for numero_linha, linha in enumerate(conteudo.splitlines(), start=1):
        texto = linha.strip()
        if not texto:
            continue

        for severidade, padrao in padroes:
            encontrado = padrao.search(texto)
            if encontrado:
                eventos.append(
                    Evento(
                        arquivo=nome_arquivo,
                        linha=numero_linha,
                        severidade=severidade,
                        termo=encontrado.group(0),
                        texto=texto,
                    )
                )
                break
    return eventos


def analisar_logs(caminho: str) -> list[Evento]:
    caminho_base = Path(caminho)
    padroes = compilar_padroes()
    eventos = []

    for nome_arquivo, conteudo in ler_arquivos(caminho_base):
        eventos.extend(analisar_conteudo(nome_arquivo, conteudo, padroes))

    return eventos


def salvar_csv(eventos: list[Evento], caminho_saida: str) -> None:
    with open(caminho_saida, "w", newline="", encoding="utf-8-sig") as arquivo_csv:
        writer = csv.DictWriter(
            arquivo_csv,
            fieldnames=["arquivo", "linha", "severidade", "termo", "texto"],
            delimiter=";",
        )
        writer.writeheader()

        for evento in eventos:
            writer.writerow(
                {
                    "arquivo": evento.arquivo,
                    "linha": evento.linha,
                    "severidade": evento.severidade,
                    "termo": evento.termo,
                    "texto": evento.texto,
                }
            )


def imprimir_relatorio(eventos: list[Evento]) -> None:
    if not eventos:
        print("Nenhuma falha, alarme ou aviso encontrado.")
        return

    resumo = {}
    for evento in eventos:
        resumo[evento.severidade] = resumo.get(evento.severidade, 0) + 1

    print("Resumo:")
    for severidade in ("CRITICO", "ERRO", "AVISO"):
        if severidade in resumo:
            print(f"- {severidade}: {resumo[severidade]}")

    print("\nEventos encontrados:")
    for evento in eventos:
        print(
            f"[{evento.severidade}] {evento.arquivo}:{evento.linha} "
            f"termo='{evento.termo}' | {evento.texto}"
        )


def main() -> None:
    parser = argparse.ArgumentParser(description="Analisa logs em arquivo .log, .txt, .csv, pasta ou .zip.")
    parser.add_argument("caminho", nargs="?", help="Caminho do log, pasta com logs ou arquivo .zip.")
    parser.add_argument("--csv", help="Opcional: salva os eventos encontrados em CSV.")
    args = parser.parse_args()

    if not args.caminho:
        print("Informe o caminho do arquivo ou pasta.")
        print('Exemplo: python analisador_logs.py "C:\\logs\\arquivo.zip" --csv resultado.csv')
        raise SystemExit(1)

    try:
        eventos = analisar_logs(args.caminho)
        imprimir_relatorio(eventos)

        if args.csv:
            salvar_csv(eventos, args.csv)
            print(f"\nCSV salvo em: {os.path.abspath(args.csv)}")

    except (FileNotFoundError, zipfile.BadZipFile) as erro:
        print(f"Erro ao carregar logs: {erro}", file=sys.stderr)
        raise SystemExit(1)


if __name__ == "__main__":
    main()
