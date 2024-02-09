import folium
from geopy.geocoders import Nominatim
from kivy.app import App
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.label import Label
from kivy.uix.button import Button
from kivy_garden.mapview import MapView, MapMarker
from kivy.clock import Clock
from kivy.uix.popup import Popup
from kivy.uix.textinput import TextInput

def get_gsm_location(modem):
    # Отправляем команду AT для получения информации о местоположении
    response = modem.sendCommand('AT+CIPGSMLOC=1,1')
    # Ответ содержит информацию о местоположении в формате LAC, Cell ID, MCC, MNC
    location_info = response.split(',')

    if len(location_info) == 4:
        return {
            'LAC': location_info[0],
            'CellID': location_info[1],
            'MCC': location_info[2],
            'MNC': location_info[3]
        }
    else:
        return None

def get_coordinates(location_info):
    # Преобразуем полученное местоположение в координаты
    mcc = location_info['MCC']
    mnc = location_info['MNC']
    lac = location_info['LAC']
    cell_id = location_info['CellID']

    # Создаем строку, которую можно использовать для запроса местоположения
    location_str = f'{mcc},{mnc},{lac},{cell_id}'

    # Запрос местоположения через Nominatim
    geolocator = Nominatim(user_agent="gsm_location_app")
    location = geolocator.reverse(location_str)

    return location.latitude, location.longitude

class DeliveryApp(App):
    def build(self):
        self.courier_location_label = Label(text="Геолокация курьера: Загружается...", color=(0, 1, 0, 1), size_hint=(1, 0.05))

        self.map_view = MapView(zoom=15, lat=55.7522200, lon=37.61556004, size_hint=(1, 0.7))
        self.courier_marker = MapMarker(lat=self.map_view.lat, lon=self.map_view.lon)
        self.map_view.add_marker(self.courier_marker)

        update_location_button = Button(text="Обновить локацию", background_color=(0, 2, 1, 1), size_hint=(1, 0.25))
        update_location_button.bind(on_press=self.update_courier_location)

        message_button = Button(text="Новое сообщение", size_hint=(None, None), size=(200, 50), pos_hint={'center_x': 0.5})
        message_button.bind(on_press=self.open_message_popup)
        message_button.bind(on_press=self.open_message_popup)
        message_button.background_color = (0, 1, 1, 1)  # Установка цвета фона кнопки
        message_button.border_radius = [25]  # Задание радиуса скругления углов

        layout = BoxLayout(orientation='vertical', spacing=5, padding=5)
        layout.add_widget(self.courier_location_label)
        layout.add_widget(self.map_view)
        layout.add_widget(update_location_button)
        layout.add_widget(message_button)

        Clock.schedule_interval(self.update_courier_location, 30)

        return layout

    def update_courier_location(self, *args):


        try:
            # Инициализируем модем
            if modem:
                modem.connect()

                # Получаем информацию о местоположении
                gsm_location = get_gsm_location(modem)

                if gsm_location:
                    # Преобразуем местоположение в координаты
                    coordinates = get_coordinates(gsm_location)

                    self.courier_location_label.text = f"Courier's Location: Lat: {coordinates[0]:.4f}, Lon: {coordinates[1]:.4f}"
                    self.courier_marker.lat = coordinates[0]
                    self.courier_marker.lon = coordinates[1]

                    # Создаем карту
                    map = folium.Map(location=coordinates, zoom_start=12)
                    # Добавляем маркер
                    folium.Marker(location=coordinates, popup='GSM Location').add_to(map)

                    # Сохраняем карту в HTML файл
                    map.save('gsm_location_map.html')

                    print("Map created and saved as 'gsm_location_map.html'.")
                else:
                    print("Unable to get GSM location information.")

        except Exception as e:
            print("Error:", str(e))
        finally:
            # Закрываем соединение с модемом
            if modem:
                modem.disconnect()

    def open_message_popup(self, *args):
        content = BoxLayout(orientation='vertical')
        text_input = TextInput(text='', multiline=False)
        content.add_widget(Label(text='Введите сообщение:'))
        content.add_widget(text_input)

        popup = Popup(title='Новое сообщение', content=content, size_hint=(None, None), size=(400, 200))
        popup.open()

if __name__ == '__main__':
    DeliveryApp().run()
