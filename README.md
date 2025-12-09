#Ricoy Ramos Julio Alexander
#952
#14/09/2025

#importamos las librerias necesarias
import pandas as pd
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from webdriver_manager.chrome import ChromeDriverManager
from bs4 import BeautifulSoup
import time
import os


def extraer(html):
    # extraccion de datos y creacion del diccionario
    data = {
        "Titulo": [],
        "Precio": [],
        "Rating": [],
        "Envio": []
    }

    soup = BeautifulSoup(html, 'html.parser')

    # exploracion de productos de Amazon
    productos = soup.find_all("div", {"data-component-type": "s-search-result"})

    for producto in productos:
        # Título
        titulo = producto.find("h2", class_="a-size-base-plus a-spacing-none a-color-base a-text-normal")
        if titulo:
            link = titulo.find("a")
            if link:
                data["Titulo"].append(link.text.strip())
            else:
                data["Titulo"].append(titulo.text.strip())
        else:
            data["Titulo"].append("Sin título")

        # Precio
        precio = producto.find("span", class_="a-price-whole")
        if precio:
            data["Precio"].append(precio.text.strip())
        else:
            data["Precio"].append("Sin precio")

        # Rating
        rating = producto.find("span", class_="a-icon-alt")
        if rating:
            data["Rating"].append(rating.text.strip())
        else:
            data["Rating"].append("Sin rating")

        # Envío (utilice IA para poder extraerlo, pero entiendo cual es su funcion
        # lo primero que realiza y se almacena en la variable de envio_elemento son los datos del envio en el html desde la
        #etiqueta div, y la clase, en amazon mx o us cambia y tenia errores al momento de extraer
        envio_elemento = producto.find("div", class_="a-row a-color-base udm-primary-delivery-message")
        if envio_elemento: # realizamos el ciclo para poder buscar los datos dentro de las etiquetas dentro de la pagina
                data["Envio"].append(envio_elemento.text.strip())
        else:
            # Si no encuentra datos de envio rellena con SIN INFORMACION
            data["Envio"].append("Sin información")

    return pd.DataFrame(data)


def navegar(producto, paginas):
    # navegacion dentro de la pagina
    s = Service(ChromeDriverManager().install())
    opc = Options()
    opc.add_argument("--window-size=1920x1080")
    navegador = webdriver.Chrome(service=s, options=opc)

    # Ir a Amazon México
    navegador.get("https://www.amazon.com.mx")
    time.sleep(3)

    # Crear la carpeta de screenshots si no existe
    if not os.path.exists('screenshots'):
        os.makedirs('screenshots')

    # Buscar el producto usando la barra de búsqueda
    try:
        # Encontrar la caja de búsqueda
        search_box = navegador.find_element(By.ID, "twotabsearchtextbox")
        search_box.clear()
        search_box.send_keys(producto)

        # Hacer clic en el botón de búsqueda
        search_button = navegador.find_element(By.ID, "nav-search-submit-button")
        search_button.click()

        time.sleep(3)
        print(f"Búsqueda realizada para: {producto}")

    except:
        # Si falla, usar URL directa como respaldo
        url = f"https://www.amazon.com.mx/s?k={producto.replace(' ', '+')}"
        navegador.get(url)
        print(f"Usando búsqueda directa: {url}")

    df_final = pd.DataFrame()

    for pag in range(paginas):
        time.sleep(3)

        # Capturar screenshot
        navegador.save_screenshot(f"screenshots/pagina_{pag + 1}.png")

        html = navegador.page_source
        df_temp = extraer(html)

        # Concatenar datos
        df_final = pd.concat([df_final, df_temp], ignore_index=True)

        # Navegar a siguiente página
        if pag < paginas - 1:
            try:
                siguiente = navegador.find_element(By.PARTIAL_LINK_TEXT, "Siguiente")
                siguiente.click()
            except:
                print("No hay mas paginas")
                break

    navegador.quit()

    # La carpeta 'datos' también debe crearse antes de intentar guardar el CSV
    if not os.path.exists('datos'):
        os.makedirs('datos')

    # Guardar CSV
    df_final.to_csv("datos/amazon_productos.csv", index=False)
    print("Datos guardados exitosamente")


if __name__ == '__main__':
    producto = "refrigeracion liquida"
    paginas = 3
    navegar(producto, paginas)
