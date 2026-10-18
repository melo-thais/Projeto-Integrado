# Procedimento para caracterização de condições atmosféricas pela reanálise ERA5

[![Abrir no Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/melo-thais/Projeto-Integrado/blob/main/atmosfera-era5/atmosfera_era5.ipynb)

Notebook para o Google Colab que obtém da reanálise ERA5 as condições atmosféricas de um lançamento já realizado: pressão, temperatura, vento, umidade e nebulosidade nos níveis de pressão que cobrem o voo. O arquivo final, em NetCDF, pode ser lido diretamente pelo RocketPy.

O procedimento é descrito no apêndice "Procedimento para caracterização de condições atmosféricas pela reanálise ERA5" do trabalho de conclusão de curso *Análise aerodinâmica de foguetes experimentais em regime de voo livre por processamento de sinais e rede neural informada por física* (Escola Politécnica da USP).

## O que o notebook faz

1. **Estima os níveis de pressão** que cobrem o voo, do trilho ao apogeu, pela atmosfera padrão (ISO 2533), com um fator de segurança sobre a altura do apogeu.
2. **Define os horários e a área**: as duas horas cheias em torno do lançamento e uma área com folga de 0,5° em torno do trilho (ajustável).
3. **Faz a requisição ao Climate Data Store (CDS)** pela API, com os mesmos campos do formulário do site.
4. **Confere o arquivo baixado**: em cada horário, verifica se há pelo menos um nível abaixo do trilho e se o nível superior fica acima do apogeu com o fator de segurança. Se uma condição falhar, acrescenta um nível no lado que falhou e refaz a requisição, até as duas serem cumpridas até um número máximo de tentativas (ajustável).
5. **Confere o relevo do modelo**: obtém o relevo do ERA5 na mesma área e verifica, nos quatro pontos da grade em torno do trilho, se os níveis usados na interpolação até o trilho estão acima dele. Abaixo do relevo, os valores dos níveis de pressão são extrapolados, e não calculados pelo modelo.
6. **Salva o arquivo final** na pasta de trabalho e mostra os valores usados na conferência.

## Antes do primeiro uso

1. Crie uma conta no [Climate Data Store](https://cds.climate.copernicus.eu), se ainda não tiver.
2. Abra as páginas [*ERA5 hourly data on pressure levels*](https://cds.climate.copernicus.eu/datasets/reanalysis-era5-pressure-levels) e [*ERA5 hourly data on single levels*](https://cds.climate.copernicus.eu/datasets/reanalysis-era5-single-levels) e aceite a licença de cada uma (aba *Download*, no fim do formulário).
3. Copie a sua chave (*Personal Access Token*) na página [How to use the CDS API](https://cds.climate.copernicus.eu/how-to-api), com a sua conta conectada: ela aparece no quadro de configuração, na linha `key:`.
4. Guarde a chave nos *Secrets* do Colab: no ícone de chave da barra lateral, adicione um segredo com o nome `CDSAPI_KEY` e a chave como valor, e ative o acesso do notebook. Assim a chave não fica escrita no notebook e não vai junto se ele for compartilhado.

Sem a chave, o notebook também funciona com `USAR_API = False` (Seção 2): ele mostra a requisição para ser feita no site e confere o arquivo enviado. Nesse caso, bastam os passos 1 e 2.

## Como usar

1. Abra o notebook no Colab pelo botão acima.
2. Preencha a célula **Entradas** com os dados do lançamento.
3. Na célula **Pasta de trabalho**, escreva em `PASTA_TRABALHO` o caminho da pasta do Google Drive onde os arquivos devem ser salvos, a partir de **Meu Drive** (por exemplo, `"TCC/ERA5"`). Se a pasta não existir, ela é criada. Se `PASTA_TRABALHO` ficar vazia, os arquivos vão para `/content` e são apagados ao fim da sessão.
4. Use *Ambiente de execução → Executar tudo*.
5. Para baixar também o arquivo final no computador, use `BAIXAR_NO_COMPUTADOR = True` na última célula.

| Entrada | Descrição |
|---|---|
| `nome` | Nome do foguete, usado no nome dos arquivos |
| `lat`, `lon` | Latitude e longitude do trilho, em graus (positivas ao norte e a leste) |
| `h_t` | Altitude do trilho em relação ao nível do mar (m) |
| `data_utc`, `hora_utc` | Data e instante do lançamento, em UTC |
| `dh_apo` | Altura do apogeu acima do trilho (m) |
| `origem_apo` | Como o apogeu foi obtido: `"barometro"` (altímetro barométrico), `"gnss"` ou `"previsto"` (só simulado, voo sem dados) |
| `margem_area` | Folga mínima da área em torno do trilho (graus; pelo menos 0,25°, padrão 0,5°) |

## Critérios adotados

- **Atmosfera padrão**: camadas da ISO 2533, as mesmas do RocketPy. A estimativa não considera a pressão e a temperatura do dia, que deslocam a altitude dos níveis. Se o deslocamento deixar parte do voo sem cobertura, a conferência acusa e um nível é acrescentado.
- **Fator `F` sobre a altura do apogeu**:
  - 1,1 para altímetro barométrico. Esses altímetros convertem pressão em altura pela atmosfera padrão, e o erro é de cerca de 4 % a cada 10 °C de desvio de temperatura (EUROCONTROL, 2014, com base no ICAO Doc 8168). O fator cobre desvios de até cerca de 25 °C.
  - 1,0 para GNSS, cuja medição não depende da pressão.
  - 2,0 para apogeu só previsto. Cada nível a mais acrescenta só alguns quilobytes, e o RocketPy interpola apenas entre os dois níveis vizinhos de cada altitude.
- **Área**: limites nos pontos da grade de 0,25° do ERA5. A interpolação até o trilho usa só o ponto vizinho de cada lado. A folga padrão de 0,5° deixa um ponto a mais, o que permite corrigir a posição do trilho em até 0,25° sem baixar os dados de novo.

O procedimento vale para voos abaixo de cerca de 48 km, o topo dos níveis de pressão do ERA5 (1 hPa).

## Saídas

- `era5_<nome>_pressoes.nc`: arquivo final, com os dados nos níveis de pressão, lido pelo RocketPy.
- `era5_<nome>_relevo.nc`: relevo do modelo (geopotencial da superfície), usado na conferência do relevo.

No RocketPy, o arquivo é lido com `Environment.set_atmospheric_model(type="Reanalysis", file=..., dictionary="ECMWF")`.

## Requisitos

No Colab, a primeira célula instala o que falta. Para rodar em um Jupyter local, instale os pacotes de `requirements.txt`. `PASTA_TRABALHO` passa a ser um caminho no próprio computador e, vazia, os arquivos ficam na pasta do notebook. A leitura da chave (Secrets) e o envio de arquivos no modo `USAR_API = False` usam ferramentas próprias do Colab e, fora dele, precisam ser adaptados.

## Referências

- ECMWF. *IFS documentation: part VI: technical and computational procedures* (CY23R4). Reading, 2003. Disponível em: <https://www.ecmwf.int/sites/default/files/elibrary/2003/13275-part-vi-technical-and-computational-procedures.pdf>.
- EUROCONTROL. *EUROCONTROL guidelines for cold temperature corrections by ATS*. Ed. 1.0. Bruxelas, 2014. Disponível em: <https://skybrary.aero/bookshelf/books/2939.pdf>.
- HERSBACH, H. et al. The ERA5 global reanalysis. *Quarterly Journal of the Royal Meteorological Society*, v. 146, n. 730, p. 1999–2049, 2020. DOI: [10.1002/qj.3803](https://doi.org/10.1002/qj.3803).
- HERSBACH, H. et al. *ERA5 hourly data on pressure levels from 1940 to present*. Copernicus Climate Change Service (C3S) Climate Data Store (CDS), 2023. DOI: [10.24381/cds.bd0915c6](https://doi.org/10.24381/cds.bd0915c6).
- HERSBACH, H. et al. *ERA5 hourly data on single levels from 1940 to present*. Copernicus Climate Change Service (C3S) Climate Data Store (CDS), 2023. DOI: [10.24381/cds.adbb2d47](https://doi.org/10.24381/cds.adbb2d47).
- INTERNATIONAL ORGANIZATION FOR STANDARDIZATION. *ISO 2533: standard atmosphere*. Genebra, 1975.
- ROCKETPY TEAM. *RocketPy documentation*. Disponível em: <https://docs.rocketpy.org>.

## Como citar

Veja o arquivo [`CITATION.cff`](../CITATION.cff) na raiz do repositório, ou use o botão *Cite this repository* do GitHub.

## Licença

MIT. Veja o arquivo [`LICENSE`](../LICENSE) na raiz do repositório.
