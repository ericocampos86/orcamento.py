from kivy.uix.boxlayout import BoxLayout
from kivy.uix.label import Label
from kivy.uix.button import Button
from kivy.uix.textinput import TextInput
from kivy.uix.popup import Popup
from kivy.uix.checkbox import CheckBox
from kivy.clock import Clock
from kivy.app import App
import json
import os
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders
import logging
from unidecode import unidecode
from kivy.logger import Logger



class Caneta:
    def __init__(self, app, material):
        self.app = app
        self.material = material.upper()
        self.processos = ["silk ", "Tampografia", "Laser"] if self.material in ["BAMBU", "METAL"] else ["silk",
                                                                                                        "Tampografia"]

    def escolher_processo(self):
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text="Escolha o processo:", size_hint_y=None, height=30)
        layout.add_widget(label)

        for processo in self.processos:
            button = Button(text=processo.strip(), size_hint_y=None, height=50)
            button.bind(on_press=lambda instance, proc=processo: self.processo_selecionado(proc.strip()))
            layout.add_widget(button)

        self.app.save_previous_screen()
        self.app.root.clear_widgets()
        self.app.root.add_widget(layout)

    def processo_selecionado(self, processo):
        processo = processo.upper().strip()
        if processo in ["SILK", "TAMPOGRAFIA", "LASER"]:
            self.parametros_orcamento(processo)
        else:
            self.app.show_error("Erro", f"Processo inválido. Por favor, escolha entre {', '.join(self.processos)}.")

    def parametros_orcamento(self, processo):
        if self.material == "METAL":
            self.app.show_input_dialog("Número de Logos", "Quantos logos?",
                                       self.processar_parametros_orcamento_sem_local, processo)
        else:
            self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_parametros_orcamento,
                                       processo)

    def processar_parametros_orcamento(self, numero_logos, processo):
        try:
            numero_logos = int(numero_logos)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de logos.")
            return

        if numero_logos > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 logos.")
            return

        if processo == "LASER":
            self.perguntar_cor_logo(processo, numero_logos)
        else:
            self.app.show_input_dialog("Cores", "Quantas cores?", self.processar_cores_orcamento, processo,
                                       numero_logos)

    def processar_parametros_orcamento_sem_local(self, numero_logos, processo):
        try:
            numero_logos = int(numero_logos)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de logos.")
            return

        if numero_logos > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 logos.")
            return

        if processo == "LASER":
            self.perguntar_cor_logo_sem_local(processo, numero_logos)
        else:
            self.app.show_input_dialog("Cores", "Quantas cores?", self.processar_cores_orcamento_sem_local, processo,
                                       numero_logos)

    def perguntar_cor_logo(self, processo, numero_logos):
        self.app.show_input_dialog("Cor do Logo", "Deseja que o logo seja escuro? (Sim/Não)", self.processar_cor_logo,
                                   processo, numero_logos)

    def perguntar_cor_logo_sem_local(self, processo, numero_logos):
        self.app.show_input_dialog("Cor do Logo", "Deseja que o logo seja escuro? (Sim/Não)",
                                   self.processar_cor_logo_sem_local, processo, numero_logos)

    def processar_cor_logo(self, cor_logo, processo, numero_logos):
        cor_logo = cor_logo.strip().lower()
        if cor_logo not in ["sim", "não", "nao"]:
            self.app.show_error("Erro", "Por favor, responda com 'Sim' ou 'Não'.")
            return

        logo_escuro = cor_logo == "sim"
        self.app.show_input_dialog("Local", "Aplicação no corpo ou no clipe?", self.processar_local_orcamento, processo,
                                   numero_logos, None, logo_escuro)

    def processar_cor_logo_sem_local(self, cor_logo, processo, numero_logos):
        cor_logo = cor_logo.strip().lower()
        if cor_logo not in ["sim", "não", "nao"]:
            self.app.show_error("Erro", "Por favor, responda com 'Sim' ou 'Não'.")
            return

        logo_escuro = cor_logo == "sim"
        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_orcamento, processo,
                                   numero_logos, None, "CORPO", logo_escuro)

    def processar_cores_orcamento(self, cores, processo, numero_logos):
        try:
            cores = int(cores)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de cores.")
            return

        self.app.show_input_dialog("Local", "Aplicação no corpo ou no clipe?", self.processar_local_orcamento, processo,
                                   numero_logos, cores)

    def processar_cores_orcamento_sem_local(self, cores, processo, numero_logos):
        try:
            cores = int(cores)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de cores.")
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_orcamento, processo,
                                   numero_logos, cores, "CORPO")

    def processar_local_orcamento(self, local, processo, numero_logos, cores, logo_escuro=False):
        local = self.app.normalizar_entrada(local)
        if local not in ["CORPO", "CLIPE"]:
            self.app.show_error("Erro", "Local inválido. Por favor, escolha entre corpo ou clipe.")
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_orcamento, processo,
                                   numero_logos, cores, local, logo_escuro)

    def processar_quantidade_orcamento(self, quantidade, processo, numero_logos, cores, local, logo_escuro=False):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        def on_bag_response(resposta):
            custo_producao = self.calcular_custo_producao(processo, numero_logos, cores, quantidade, logo_escuro)
            unitario = custo_producao / quantidade if quantidade != 0 else 0
            self.app.salvar_orcamento(self.material, processo, numero_logos, cores, local, quantidade, custo_producao,
                                      unitario)
            self.app.mostrar_relatorio(self.material, processo, numero_logos, cores, local, quantidade, custo_producao,
                                       unitario)

        self.app.show_input_dialog("Saquinho", "Deseja que as canetas sejam colocadas no saquinho? (Sim/Não)",
                                   on_bag_response)

    def calcular_custo_producao(self, processo, numero_logos, cores, quantidade, logo_escuro):
        custo_producao = 0
        if self.material in ["PLASTICO", "PAPELAO"]:
            if processo == "SILK":
                if quantidade <= 1000:
                    custo_producao = 100 * numero_logos * cores
                else:
                    custo_producao = quantidade * 0.10 * numero_logos * cores
            elif processo == "TAMPOGRAFIA":
                if quantidade <= 1000:
                    custo_producao = (150 + 50) * numero_logos * cores
                else:
                    custo_producao = ((quantidade * 0.15) + 50) * numero_logos * cores

        if self.material == "BAMBU":
            if processo == "SILK":
                if quantidade <= 1000:
                    custo_producao = 100 * numero_logos * cores
                else:
                    custo_producao = quantidade * 0.10 * numero_logos * cores
            elif processo == "TAMPOGRAFIA":
                if quantidade <= 1000:
                    custo_producao = (150 + 50) * numero_logos * cores
                else:
                    custo_producao = ((quantidade * 0.15) + 50) * numero_logos * cores
            elif processo == "LASER":
                if quantidade <= 267:
                    custo_producao = 120 * numero_logos
                else:
                    custo_producao = quantidade * 0.45 * numero_logos

        if self.material == "METAL":
            if processo == "SILK":
                if quantidade <= 1000:
                    custo_producao = 200 * numero_logos * cores
                else:
                    custo_producao = quantidade * 0.20 * numero_logos * cores
            elif processo == "TAMPOGRAFIA":
                if quantidade <= 1000:
                    custo_producao = (200 + 50) * numero_logos * cores
                else:
                    custo_producao = ((quantidade * 0.20) + 50) * numero_logos * cores
            elif processo == "LASER":
                if quantidade <= 400:
                    custo_producao = 120 * numero_logos
                else:
                    custo_producao = quantidade * 0.30 * numero_logos

        if processo == "LASER" and logo_escuro:
            custo_producao *= 2

        return custo_producao


class Lapis:
    def __init__(self, app):
        self.app = app
        self.material = "LAPIS"
        self.processos = ["silk screen", "Tampografia"]

    def escolher_processo(self):
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text="Escolha o processo:", size_hint_y=None, height=30)
        layout.add_widget(label)

        for processo in self.processos:
            button = Button(text=processo, size_hint_y=None, height=50)
            button.bind(on_press=lambda instance, proc=processo: self.processo_selecionado(proc))
            layout.add_widget(button)

        self.app.save_previous_screen()  # Salva a tela anterior
        self.app.root.clear_widgets()
        self.app.root.add_widget(layout)

    def processo_selecionado(self, processo):
        if processo in ["silk screen", "Tampografia"]:
            self.parametros_orcamento(processo)
        else:
            self.app.show_error("Erro", f"Processo inválido. Por favor, escolha entre {', '.join(self.processos)}.")
            # A tela será restaurada automaticamente após o popup ser fechado

    def parametros_orcamento(self, processo):
        self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_parametros_orcamento, processo)

    def processar_parametros_orcamento(self, numero_logos, processo):
        try:
            numero_logos = int(numero_logos)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de logos.")
            return

        if processo == "Silk" and numero_logos > 1:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 1 logo.")
            self.app.contato_vendedor()
            return

        self.app.show_input_dialog("Cores", "Quantas cores?", self.processar_cores_orcamento, processo, numero_logos)

    def processar_cores_orcamento(self, cores, processo, numero_logos):
        try:
            cores = int(cores)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de cores.")
            return

        if processo == "silk screen" and cores > 1:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 1 cor.")
            self.app.contato_vendedor()
            return
        elif processo == "Tampografia" and cores > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 cores.")
            self.app.contato_vendedor()
            return

        self.parametros_orcamento_final(processo, numero_logos, cores)

    def parametros_orcamento_final(self, processo, numero_logos, cores):
        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_orcamento, processo,
                                   numero_logos, cores)

    def processar_quantidade_orcamento(self, quantidade, processo, numero_logos, cores):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        custo_producao = self.calcular_custo_producao(processo, numero_logos, cores, quantidade)
        unitario = custo_producao / quantidade

        if None in [custo_producao, unitario]:
            self.app.root.quit()
        else:
            self.app.salvar_orcamento(self.material, processo, numero_logos, cores, None, quantidade, custo_producao,
                                      unitario)
            self.app.mostrar_relatorio(self.material, processo, numero_logos, cores, None, quantidade, custo_producao,
                                       unitario)

    def calcular_custo_producao(self, processo, numero_logos, cores, quantidade):
        custo_producao = 0
        if self.material == "LAPIS":
            if processo == "silk screen":
                if quantidade <= 1000:
                    custo_producao = 100 * numero_logos * cores
                else:
                    custo_producao = quantidade * 0.10 * numero_logos * cores
            elif processo == "Tampografia":
                if quantidade <= 1000:
                    custo_producao = (150 + 50) * numero_logos * cores
                else:
                    custo_producao = ((quantidade * 0.15) + 50) * numero_logos * cores
        return custo_producao


class Bolsa:
    def __init__(self, app):
        self.app = app
        self.material = "BOLSA"
        self.processos = ["silk screen"]

    def escolher_processo(self):
        self.app.show_info("Processo", "Orçamento para bolsas é baseado no processo em silk screen.")
        self.parametros_orcamento("silk screen")

    def parametros_orcamento(self, processo):
        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'", self.processar_impressao,
                                   processo)

    def processar_impressao(self, impressao, processo):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento(processo)

        self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_numero_logos, processo,
                                   impressao)

    def processar_numero_logos(self, numero_logos, processo, impressao):
        try:
            numero_logos = int(numero_logos)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de logos.")
            return

        if numero_logos > 4:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 4 logos.")
            self.app.contato_vendedor()
            return

        self.app.show_input_dialog("Cores", "Quantas cores?", self.processar_cores, processo, impressao, numero_logos)

    def processar_cores(self, cores, processo, impressao, numero_logos):
        try:
            cores = int(cores)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de cores.")
            return

        if cores > 4:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 4 cores.")
            self.app.contato_vendedor()
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade, processo, impressao,
                                   numero_logos, cores)

    def processar_quantidade(self, quantidade, processo, impressao, numero_logos, cores):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        if impressao == "FRENTEEVERSO":
            quantidade *= 2

        custo_producao = self.calcular_custo_producao(processo, numero_logos, cores, quantidade)
        unitario = custo_producao / quantidade

        if None in [custo_producao, unitario]:
            self.app.root.quit()
        else:
            self.app.salvar_orcamento(self.material, processo, numero_logos, cores, impressao, quantidade,
                                      custo_producao, unitario)
            self.app.mostrar_relatorio(self.material, processo, numero_logos, cores, impressao, quantidade,
                                       custo_producao, unitario)

    def calcular_custo_producao(self, processo, numero_logos, cores, quantidade):
        custo_producao = 0
        if self.material == "BOLSA":
            if processo == "silk screen":
                if quantidade <= 100:
                    custo_producao = 180 * numero_logos * cores
                elif quantidade <= 200:
                    custo_producao = 300 * numero_logos * cores
                else:
                    custo_producao = quantidade * 1.5 * numero_logos * cores
        return custo_producao


class Necesseries:
    def __init__(self, app):
        self.app = app
        self.material = "NECESSERIES"
        self.processos = ["Silk", "Digital"]

    def escolher_processo(self):
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text="Escolha o processo:", size_hint_y=None, height=30)
        layout.add_widget(label)

        for processo in self.processos:
            button = Button(text=processo, size_hint_y=None, height=50)
            button.bind(on_press=lambda instance, proc=processo: self.processo_selecionado(proc))
            layout.add_widget(button)

        self.app.save_previous_screen()  # Salva a tela anterior
        self.app.root.clear_widgets()
        self.app.root.add_widget(layout)

    def processo_selecionado(self, processo):
        if processo in ["Silk", "Digital"]:
            if processo == "Silk":
                self.parametros_orcamento_silk(processo)
            elif processo == "Digital":
                self.parametros_orcamento_digital(processo)
        else:
            self.app.show_error("Erro", f"Processo inválido. Por favor, escolha entre {', '.join(self.processos)}.")

    def parametros_orcamento_silk(self, processo):
        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'",
                                   self.processar_impressao_silk, processo)

    def processar_impressao_silk(self, impressao, processo):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento_silk(processo)

        self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_numero_logos_silk, processo,
                                   impressao)

    def processar_numero_logos_silk(self, numero_logos, processo, impressao):
        try:
            numero_logos = int(numero_logos)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de logos.")
            return

        if numero_logos > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 logos.")
            return

        self.app.show_input_dialog("Cores", "Quantas cores?", self.processar_cores_silk, processo, impressao,
                                   numero_logos)

    def processar_cores_silk(self, cores, processo, impressao, numero_logos):
        try:
            cores = int(cores)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de cores.")
            return

        if cores > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 cores.")
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_silk, processo,
                                   impressao, numero_logos, cores)

    def processar_quantidade_silk(self, quantidade, processo, impressao, numero_logos, cores):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        if impressao == "FRENTEEVERSO":
            quantidade *= 2

        custo_producao = self.calcular_custo_producao_silk(processo, numero_logos, cores, quantidade)
        unitario = custo_producao / quantidade if quantidade != 0 else 0
        self.app.salvar_orcamento(self.material, processo, numero_logos, cores, impressao, quantidade, custo_producao,
                                  unitario)
        self.app.mostrar_relatorio(self.material, processo, numero_logos, cores, impressao, quantidade, custo_producao,
                                   unitario)

    def calcular_custo_producao_silk(self, processo, numero_logos, cores, quantidade):
        custo_producao = 0
        if processo == "Silk":
            if quantidade <= 150:
                custo_producao = 180 * numero_logos * cores
            elif quantidade <= 200:
                custo_producao = 250 * numero_logos * cores
            elif quantidade <= 300:
                custo_producao = 300 * numero_logos * cores
            else:
                custo_producao = quantidade * 1 * numero_logos * cores
        return custo_producao

    def parametros_orcamento_digital(self, processo):
        self.app.show_input_dialog("Altura do Logo", "Digite a altura do logo (mm):",
                                   self.processar_altura_logo_digital, processo)

    def processar_altura_logo_digital(self, alt, processo):
        try:
            alt = float(alt)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a altura.")
            return

        self.app.show_input_dialog("Comprimento do Logo", "Digite o comprimento do logo (mm):",
                                   self.processar_comprimento_logo_digital, processo, alt)

    def processar_comprimento_logo_digital(self, comp, processo, alt):
        try:
            comp = float(comp)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o comprimento.")
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_digital, processo, alt,
                                   comp)

    def processar_quantidade_digital(self, quantidade, processo, alt, comp):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        self.app.processar_processo_digital(quantidade, alt, comp)


class Squize:
    def __init__(self, app):
        self.app = app
        self.material = "Squize"
        self.processos = ["Silk", "Digital", "Laser"]

    def escolher_processo(self):
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text="Escolha o processo:", size_hint_y=None, height=30)
        layout.add_widget(label)

        for processo in self.processos:
            button = Button(text=processo, size_hint_y=None, height=50)
            button.bind(on_press=lambda instance, proc=processo: self.processo_selecionado(proc))
            layout.add_widget(button)

        self.app.save_previous_screen()  # Salva a tela anterior
        self.app.root.clear_widgets()
        self.app.root.add_widget(layout)

    def processo_selecionado(self, processo):
        if processo in ["Silk", "Digital", "Laser"]:
            if processo == "Laser":
                self.parametros_orcamento_laser(processo)
            elif processo == "Digital":
                self.parametros_orcamento_digital(processo)
            else:
                self.parametros_orcamento_silk(processo)
        else:
            self.app.show_error("Erro", f"Processo inválido. Por favor, escolha entre {', '.join(self.processos)}.")

    def parametros_orcamento_silk(self, processo):
        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'",
                                   self.processar_impressao_silk, processo)

    def processar_impressao_silk(self, impressao, processo):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento_silk(processo)

        self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_numero_logos_silk, processo,
                                   impressao)

    def processar_numero_logos_silk(self, numero_logos, processo, impressao):
        try:
            numero_logos = int(numero_logos)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de logos.")
            return

        if numero_logos > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 logos.")
            return

        self.app.show_input_dialog("Cores", "Quantas cores?", self.processar_cores_silk, processo, impressao,
                                   numero_logos)

    def processar_cores_silk(self, cores, processo, impressao, numero_logos):
        try:
            cores = int(cores)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de cores.")
            return

        if cores > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 cores.")
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_silk, processo,
                                   impressao, numero_logos, cores)

    def processar_quantidade_silk(self, quantidade, processo, impressao, numero_logos, cores):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        if impressao == "FRENTEEVERSO":
            quantidade *= 2

        custo_producao = self.calcular_custo_producao_silk(processo, numero_logos, cores, quantidade)
        unitario = custo_producao / quantidade if quantidade != 0 else 0
        self.app.salvar_orcamento(self.material, processo, numero_logos, cores, impressao, quantidade, custo_producao,
                                  unitario)
        self.app.mostrar_relatorio(self.material, processo, numero_logos, cores, impressao, quantidade, custo_producao,
                                   unitario)

    def calcular_custo_producao_silk(self, processo, numero_logos, cores, quantidade):
        custo_producao = 0
        if processo == "Silk":
            if quantidade <= 150:
                custo_producao = 180 * numero_logos * cores
            elif quantidade <= 180:
                custo_producao = 250 * numero_logos * cores
            elif quantidade <= 300:
                custo_producao = 300 * numero_logos * cores
            else:
                custo_producao = quantidade * numero_logos * cores
        return custo_producao

    def parametros_orcamento_laser(self, processo):
        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'",
                                   self.processar_impressao_laser, processo)

    def processar_impressao_laser(self, impressao, processo):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento_laser(processo)

        self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_numero_logos_laser, processo,
                                   impressao)

    def processar_numero_logos_laser(self, numero_logos, processo, impressao):
        try:
            numero_logos = int(numero_logos)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de logos.")
            return

        if numero_logos > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 logos.")
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_laser, processo,
                                   impressao, numero_logos)

    def processar_quantidade_laser(self, quantidade, processo, impressao, numero_logos):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        self.app.show_yesno_dialog("Nomes Diferentes", "Deseja adicionar nomes diferentes ao logo?",
                                   self.processar_nomes_diferentes, processo, impressao, numero_logos, quantidade)

    def processar_nomes_diferentes(self, adicionar_nomes, processo, impressao, numero_logos, quantidade):
        if adicionar_nomes:
            self.app.show_input_dialog("Quantidade de Nomes", "Quantos nomes diferentes?",
                                       self.finalizar_parametros_orcamento_laser, processo, impressao, numero_logos,
                                       quantidade)
        else:
            self.finalizar_parametros_orcamento_laser(0, processo, impressao, numero_logos, quantidade)

    def finalizar_parametros_orcamento_laser(self, quantidade_nomes, processo, impressao, numero_logos, quantidade):
        try:
            quantidade_nomes = int(quantidade_nomes)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade de nomes.")
            return

        custo_nomes_diferentes = quantidade_nomes * 1  # Ajuste o valor conforme necessário
        custo_producao = self.calcular_custo_producao_laser(processo, numero_logos, quantidade,
                                                            impressao) + custo_nomes_diferentes
        unitario = custo_producao / quantidade if quantidade != 0 else 0
        self.app.salvar_orcamento(self.material, processo, numero_logos, 1, impressao, quantidade, custo_producao,
                                  unitario)
        self.app.mostrar_relatorio(self.material, processo, numero_logos, 1, impressao, quantidade, custo_producao,
                                   unitario)

    def calcular_custo_producao_laser(self, processo, numero_logos, quantidade, impressao):
        multiplicador = 2 if impressao == "FRENTEEVERSO" else 1
        custo_producao = 0
        if quantidade <= 160:
            custo_producao = 180 * numero_logos * multiplicador
        elif quantidade <= 200:
            custo_producao = 230 * numero_logos * multiplicador
        elif quantidade <= 250:
            custo_producao = 250 * numero_logos * multiplicador
        else:
            custo_producao = quantidade * numero_logos * multiplicador

        return custo_producao

    def parametros_orcamento_digital(self, processo):
        self.app.show_input_dialog("Altura do Logo", "Digite a altura do logo (mm):",
                                   self.processar_altura_logo_digital, processo)

    def processar_altura_logo_digital(self, alt, processo):
        try:
            alt = float(alt)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a altura.")
            return

        self.app.show_input_dialog("Comprimento do Logo", "Digite o comprimento do logo (mm):",
                                   self.processar_comprimento_logo_digital, processo, alt)

    def processar_comprimento_logo_digital(self, comp, processo, alt):
        try:
            comp = float(comp)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o comprimento.")
            return

        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'",
                                   self.processar_impressao_digital, processo, alt, comp)

    def processar_impressao_digital(self, impressao, processo, alt, comp):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento_digital(processo)

        numero_logos = 1
        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_digital, processo,
                                   impressao, alt, comp, numero_logos)

    def processar_quantidade_digital(self, quantidade, processo, impressao, alt, comp, numero_logos):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        if impressao == "FRENTEEVERSO":
            quantidade *= 2

        custo_producao = self.calcular_custo_producao_digital(processo, alt, comp, quantidade, impressao, numero_logos)
        unitario = custo_producao / quantidade if quantidade != 0 else 0
        self.app.salvar_orcamento(self.material, processo, numero_logos, 1, impressao, quantidade, custo_producao,
                                  unitario)
        self.app.mostrar_relatorio(self.material, processo, numero_logos, 1, impressao, quantidade, custo_producao,
                                   unitario)

    def calcular_custo_producao_digital(self, processo, alt, comp, quantidade, impressao, numero_logos):
        entrada_minima_dtf = 30  # Exemplo de valor, ajuste conforme necessário
        multiplicador = 2 if impressao == "FRENTEEVERSO" else 1

        self.app.show_info("Produção Digital",
                           "Lembre-se que para produção digital, o seu produto terá que ser analisado!")

        medidas_alt = alt + (alt * 33.33 / 100)  # soma da medida do logo mais espaço entre logo
        logo_vertical = 400 / medidas_alt  # quantidade de logos na vertical para uma folha

        medidas_comp = comp + (comp * 33.33 / 100)  # soma da medida do logo na horizontal
        logo_horizontal = 300 / medidas_comp

        total_de_logos = logo_horizontal * logo_vertical  # total de logos em uma folha
        folhas = max(1, (quantidade + total_de_logos - 1) // total_de_logos)

        valor_folha = (folhas * 20) * 2

        if quantidade <= 100:
            custo_logo_unidade = 2
            custo_producao = (quantidade * custo_logo_unidade) + entrada_minima_dtf + valor_folha
        else:
            custo_logo_unidade = 1.5
            custo_producao = quantidade * custo_logo_unidade

        custo_producao = round(custo_producao, 2)

        # Exibir informações sobre folhas e logos
        self.app.show_info("Detalhes de Impressão Digital",
                           f"Folhas necessárias: {folhas}\nTotal de logos por folha: {int(total_de_logos)}")

        return custo_producao


class Agenda:
    def __init__(self, app):
        self.app = app
        self.material = "Agenda"
        self.processos = ["Silk", "Digital"]

    def escolher_processo(self):
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text="Escolha o processo:", size_hint_y=None, height=30)
        layout.add_widget(label)

        for processo in self.processos:
            button = Button(text=processo, size_hint_y=None, height=50)
            button.bind(on_press=lambda instance, proc=processo: self.processo_selecionado(proc))
            layout.add_widget(button)

        self.app.save_previous_screen()  # Salva a tela anterior
        self.app.root.clear_widgets()
        self.app.root.add_widget(layout)

    def processo_selecionado(self, processo):
        if processo in ["Silk", "Digital"]:
            if processo == "Silk":
                self.parametros_orcamento_silk(processo)
            elif processo == "Digital":
                self.parametros_orcamento_digital(processo)
        else:
            self.app.show_error("Erro", f"Processo inválido. Por favor, escolha entre {', '.join(self.processos)}.")

    def parametros_orcamento_silk(self, processo):
        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'",
                                   self.processar_impressao_silk, processo)

    def processar_impressao_silk(self, impressao, processo):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento_silk(processo)

        self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_numero_logos_silk, processo,
                                   impressao)

    def processar_numero_logos_silk(self, numero_logos, processo, impressao):
        try:
            numero_logos = int(numero_logos)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de logos.")
            return

        if numero_logos > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 logos.")
            return

        self.app.show_input_dialog("Cores", "Quantas cores?", self.processar_cores_silk, processo, impressao,
                                   numero_logos)

    def processar_cores_silk(self, cores, processo, impressao, numero_logos):
        try:
            cores = int(cores)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o número de cores.")
            return

        if cores > 2:
            self.app.show_info("Falar com Vendedor",
                               "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 cores.")
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_silk, processo,
                                   impressao, numero_logos, cores)

    def processar_quantidade_silk(self, quantidade, processo, impressao, numero_logos, cores):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        if impressao == "FRENTEEVERSO":
            quantidade *= 2

        custo_producao = self.calcular_custo_producao_silk(processo, numero_logos, cores, quantidade)
        unitario = custo_producao / quantidade if quantidade != 0 else 0
        self.app.salvar_orcamento(self.material, processo, numero_logos, cores, impressao, quantidade, custo_producao,
                                  unitario)
        self.app.mostrar_relatorio(self.material, processo, numero_logos, cores, impressao, quantidade, custo_producao,
                                   unitario)

    def calcular_custo_producao_silk(self, processo, numero_logos, cores, quantidade):
        custo_producao = 0
        if processo == "Silk":
            if quantidade <= 150:
                custo_producao = 180 * numero_logos * cores
            elif quantidade <= 180:
                custo_producao = 230
            elif quantidade <= 230:
                custo_producao = 250 * numero_logos * cores
            elif quantidade <= 250:
                custo_producao = 250 * numero_logos * cores
            else:
                custo_producao = quantidade * 1 * numero_logos * cores
        return custo_producao

    def parametros_orcamento_digital(self, processo):
        self.app.show_input_dialog("Altura do Logo", "Digite a altura do logo (mm):",
                                   self.processar_altura_logo_digital, processo)

    def processar_altura_logo_digital(self, alt, processo):
        try:
            alt = float(alt)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a altura.")
            return

        self.app.show_input_dialog("Comprimento do Logo", "Digite o comprimento do logo (mm):",
                                   self.processar_comprimento_logo_digital, processo, alt)

    def processar_comprimento_logo_digital(self, comp, processo, alt):
        try:
            comp = float(comp)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para o comprimento.")
            return

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_digital, processo, alt,
                                   comp)

    def processar_quantidade_digital(self, quantidade, processo, alt, comp):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        self.app.processar_processo_digital(quantidade, alt, comp)


class Chaveiro:
    def __init__(self, app, material):
        self.app = app
        self.material = material.upper()
        self.processos = ["Silk", "Tampografia", "Laser", "Digital"] if self.material in ["BAMBU", "METAL", "DIGITAL"] else ["Silk", "Tampografia", "DIGITAL"]

    def escolher_processo(self):
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text="Escolha o processo:", size_hint_y=None, height=30)
        layout.add_widget(label)

        for processo in self.processos:
            button = Button(text=processo.strip(), size_hint_y=None, height=50)
            button.bind(on_press=lambda instance, proc=processo: self.processo_selecionado(proc.strip()))
            layout.add_widget(button)

        self.app.save_previous_screen()
        self.app.root.clear_widgets()
        self.app.root.add_widget(layout)

    def processo_selecionado(self, processo):
        processo = processo.upper().strip()
        if processo in ["SILK", "TAMPOGRAFIA"]:
            self.parametros_orcamento(processo)
        elif processo == "DIGITAL":
            self.parametros_orcamento_digital(processo)
        elif processo == "LASER":
            self.parametros_orcamento_laser(processo)
        else:
            self.app.show_error(f"Processo inválido. Por favor, escolha entre {', '.join(self.processos)}.")
            self.escolher_processo()

    def parametros_orcamento(self, processo):
        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'", self.processar_impressao, processo)

    def processar_impressao(self, impressao, processo):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento(processo)

        self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_numero_logos, processo, impressao)

    def processar_numero_logos(self, numero_logos, processo, impressao):
        try:
            numero_logos = int(numero_logos)
            if numero_logos > 2:
                self.app.show_info("Falar com Vendedor", "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 logos.")
                self.app.contato_vendedor()
                return
        except ValueError:
            self.app.show_error("Número de logos inválido. Por favor, insira um número.")
            return self.parametros_orcamento(processo)

        self.app.show_input_dialog("Cores", "Quantas cores?", self.processar_cores, processo, impressao, numero_logos)

    def processar_cores(self, cores, processo, impressao, numero_logos):
        try:
            cores = int(cores)
        except ValueError:
            self.app.show_error("Número de cores inválido. Por favor, insira um número.")
            return self.parametros_orcamento(processo)

        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade, processo, impressao, numero_logos, cores)

    def processar_quantidade(self, quantidade, processo, impressao, numero_logos, cores):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Quantidade inválida. Por favor, insira um número.")
            return self.parametros_orcamento(processo)

        if impressao == "FRENTEEVERSO":
            quantidade *= 2

        custo_producao = self.calcular_custo_producao(processo, numero_logos, cores, quantidade, impressao)
        unitario = custo_producao / quantidade

        if None in [custo_producao, unitario]:
            self.app.root.quit()
        else:
            self.app.salvar_orcamento(self.material, processo, numero_logos, cores, impressao, quantidade, custo_producao, unitario)

    def parametros_orcamento_digital(self, processo):
        self.app.show_input_dialog("Altura do Logo", "Digite a altura do logo (mm):", self.processar_altura_logo_digital, processo)

    def processar_altura_logo_digital(self, alt, processo):
        try:
            alt = float(alt)
        except ValueError:
            self.app.show_error("Altura inválida. Por favor, insira um número.")
            return self.parametros_orcamento_digital(processo)

        self.app.show_input_dialog("Comprimento do Logo", "Digite o comprimento do logo (mm):", self.processar_comprimento_logo_digital, processo, alt)

    def processar_comprimento_logo_digital(self, comp, processo, alt):
        try:
            comp = float(comp)
        except ValueError:
            self.app.show_error("Comprimento inválido. Por favor, insira um número.")
            return self.parametros_orcamento_digital(processo)

        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'", self.processar_impressao_digital, processo, alt, comp)

    def processar_impressao_digital(self, impressao, processo, alt, comp):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento_digital(processo)

        numero_logos = 1
        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_digital, processo, impressao, alt, comp, numero_logos)

    def processar_quantidade_digital(self, quantidade, processo, impressao, alt, comp, numero_logos):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Quantidade inválida. Por favor, insira um número.")
            return self.parametros_orcamento_digital(processo)

        custo_producao = self.calcular_custo_producao_digital(processo, alt, comp, quantidade, impressao, numero_logos)
        unitario = custo_producao / quantidade

        if None in [custo_producao, unitario]:
            self.app.root.quit()
        else:
            self.app.salvar_orcamento(self.material, processo, numero_logos, 1, impressao, quantidade, custo_producao, unitario)

    def parametros_orcamento_laser(self, processo):
        self.app.show_input_dialog("Impressão", "Escolha entre 'Frente' ou 'Frente e Verso'", self.processar_impressao_laser, processo)

    def processar_impressao_laser(self, impressao, processo):
        impressao = self.app.normalizar_entrada(impressao)
        if impressao not in ["FRENTE", "FRENTEEVERSO"]:
            self.app.show_error("Opção inválida. Por favor, escolha entre 'Frente' ou 'Frente e Verso'.")
            return self.parametros_orcamento_laser(processo)

        self.app.show_input_dialog("Número de Logos", "Quantos logos?", self.processar_numero_logos_laser, processo, impressao)

    def processar_numero_logos_laser(self, numero_logos, processo, impressao):
        try:
            numero_logos = int(numero_logos)
            if numero_logos > 2:
                self.app.show_info("Falar com Vendedor", "Por favor, entre em contato com o vendedor para um orçamento detalhado com mais de 2 logos.")
                self.app.contato_vendedor()
                return
        except ValueError:
            self.app.show_error("Número de logos inválido. Por favor, insira um número.")
            return self.parametros_orcamento_laser(processo)

        cores = 1
        self.app.show_input_dialog("Quantidade", "Qual a quantidade?", self.processar_quantidade_laser, processo, impressao, numero_logos, cores)

    def processar_quantidade_laser(self, quantidade, processo, impressao, numero_logos, cores):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Quantidade inválida. Por favor, insira um número.")
            return self.parametros_orcamento_laser(processo)

        nomes_diferentes = self.app.show_yesno_dialog("Nomes Diferentes", "Deseja adicionar nomes diferentes ao logo?")
        custo_nomes_diferentes = 0

        if nomes_diferentes:
            self.app.show_input_dialog("Quantidade de Nomes", "Quantos nomes diferentes?", self.processar_quantidade_nomes_diferentes, processo, impressao, numero_logos, cores, quantidade)
        else:
            custo_producao = self.calcular_custo_producao_laser(processo, numero_logos, cores, quantidade, impressao, custo_nomes_diferentes)
            unitario = custo_producao / quantidade

            if None in [custo_producao, unitario]:
                self.app.root.quit()
            else:
                self.app.salvar_orcamento(self.material, processo, numero_logos, cores, impressao, quantidade, custo_producao, unitario)

    def calcular_custo_producao(self, processo, numero_logos, cores, quantidade, impressao):
        multiplicador = 2 if impressao == "FRENTEEVERSO" else 1
        custo_producao = 0
        try:
            if self.material == ["METAL","BAMBU"] and processo == "SILK":
                if quantidade <= 300:
                    custo_producao = 180 * numero_logos * cores * multiplicador
                else:
                    custo_producao = 0.6 * quantidade * numero_logos * cores * multiplicador

            elif self.material == ["METAL","BAMBU"] and processo == "TAMPOGRAFIA":
                if quantidade <= 300:
                    custo_producao = (180 + 50) * numero_logos * cores * multiplicador
                else:
                    custo_producao = ((0.6 * quantidade) + 50) * numero_logos * cores * multiplicador

            return custo_producao
        except Exception as e:
            self.app.show_error(f"Erro no cálculo de custo: {e}")
            return None

    def processar_quantidade_digital(self, quantidade, processo, alt, comp):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.app.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        self.app.processar_processo_digital(quantidade, alt, comp)

    def calcular_custo_producao_laser(self, processo, numero_logos, quantidade, impressao):
        multiplicador = 2 if impressao == "FRENTEEVERSO" else 1
        if quantidade <= 300:
            custo_producao = 180 * numero_logos * multiplicador
        else:
            custo_producao = quantidade * 0.60 * numero_logos * multiplicador
        return custo_producao


class AppOrcamento(App):

    def build(self):
        self.root = BoxLayout(orientation='vertical')
        self.orcamentos = []
        self.user_data_file = 'user_data.json'
        self.remember_me_file = 'remember_me.json'
        self.previous_screen = []
        self.is_final_report = False
        self.current_user = None
        self.users_data = self.load_users_data()
        self.digital_print_details = ""  # Variável para armazenar detalhes de impressão digital

        remembered_user = self.load_remembered_user()
        if remembered_user:
            self.show_login_screen(remembered_user)
        elif self.is_user_registered():
            self.show_login_screen()
        else:
            self.show_registration_screen()

        return self.root

    def is_user_registered(self):
        return len(self.users_data) > 0

    def load_users_data(self):
        if os.path.exists(self.user_data_file):
            with open(self.user_data_file, 'r') as f:
                return json.load(f)
        else:
            return {}

    def save_users_data(self):
        with open(self.user_data_file, 'w') as f:
            json.dump(self.users_data, f)

    def show_login_screen(self, remembered_user=None):
        self.root.clear_widgets()

        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)

        label = Label(text="Login", font_size='20sp')
        layout.add_widget(label)

        self.username_input = TextInput(hint_text="Usuário", multiline=False)
        self.password_input = TextInput(hint_text="Senha", multiline=False, password=True)
        self.remember_me = CheckBox()
        remember_me_label = Label(text="Manter login", size_hint_y=None, height=50)

        if remembered_user:
            self.username_input.text = remembered_user.get('username', '')
            self.password_input.text = remembered_user.get('password', '')
            self.remember_me.active = True

        layout.add_widget(self.username_input)
        layout.add_widget(self.password_input)
        layout.add_widget(self.remember_me)
        layout.add_widget(remember_me_label)

        login_button = Button(text="Login", on_press=self.validate_login)
        layout.add_widget(login_button)

        register_button = Button(text="Cadastrar", on_press=self.show_registration_screen)
        layout.add_widget(register_button)

        self.root.add_widget(layout)

    def show_registration_screen(self, instance=None):
        self.root.clear_widgets()

        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)

        label = Label(text="Cadastro", font_size='20sp')
        layout.add_widget(label)

        self.username_input = TextInput(hint_text="Usuário", multiline=False)
        self.password_input = TextInput(hint_text="Senha", multiline=False, password=True)
        self.email_input = TextInput(hint_text="Email", multiline=False)
        self.cpf_input = TextInput(hint_text="CPF", multiline=False)

        layout.add_widget(self.username_input)
        layout.add_widget(self.password_input)
        layout.add_widget(self.email_input)
        layout.add_widget(self.cpf_input)

        register_button = Button(text="Registrar", on_press=self.register_user)
        layout.add_widget(register_button)

        self.root.add_widget(layout)

    def register_user(self, instance):
        username = self.username_input.text
        if username in self.users_data:
            self.show_error("Erro", "Usuário já registrado.")
            return

        user_data = {
            'password': self.password_input.text,
            'email': self.email_input.text,
            'cpf': self.cpf_input.text
        }
        self.users_data[username] = user_data
        self.save_users_data()

        self.show_info("Registro", "Usuário registrado com sucesso! Agora você pode fazer login.")
        self.show_login_screen()

    def validate_login(self, instance):
        username = self.username_input.text
        password = self.password_input.text

        if username in self.users_data and self.users_data[username]['password'] == password:
            self.current_user = username
            if self.remember_me.active:
                self.remember_user()
            self.show_program(instance)
        else:
            self.show_error("Erro", "Usuário ou senha incorretos.")

    def remember_user(self):
        with open(self.remember_me_file, 'w') as f:
            json.dump({'username': self.current_user, 'password': self.users_data[self.current_user]['password']}, f)

    def load_remembered_user(self):
        if os.path.exists(self.remember_me_file):
            with open(self.remember_me_file, 'r') as f:
                data = json.load(f)
            if 'username' in data and 'password' in data:
                return data
        return None

    def create_main_frame(self):
        self.label_titulo = Label(text="Clique no ícone para iniciar o programa:", font_size='20sp')
        self.root.add_widget(self.label_titulo)
        self.create_logo_button()

    def create_logo_button(self):
        logo_button = Button(size_hint=(None, None), size=(200, 200),
                             background_normal=r"C:\Users\Cris\Documents\Downloads\LOGO GRANGRAFICO.png",
                             on_press=self.show_program)
        self.root.add_widget(logo_button)

    def show_program(self, instance):
        self.root.clear_widgets()
        self.label_titulo = Label(text="Escolha o produto para orçamento:", font_size='20sp')
        self.root.add_widget(self.label_titulo)
        self.create_product_buttons()

    def create_product_buttons(self):
        produtos = ["CANETAS - LAPIS", "BOLSAS - SACOLAS", "NECESSERIES - MOCHILAS", "SQUEZE - COPOS - GARRAFAS",
                    "AGENDAS - BLOCOS - PORTA JOIAS", "CHAVEIRO - PLAQUINHAS", "NENHUMA DAS ALTERNATIVAS"]
        for produto in produtos:
            button = Button(text=produto, size_hint_y=None, height=50)
            button.bind(on_press=lambda instance, prod=produto: self.escolher_produto(prod))
            self.root.add_widget(button)

    def show_input_dialog(self, title, message, callback, *args):
        self.save_previous_screen()  # Salva a tela anterior
        layout = BoxLayout(orientation='vertical', padding=10)
        label = Label(text=message, size_hint_y=None, height=30)
        input_field = TextInput(multiline=False, size_hint_y=None, height=30)
        layout.add_widget(label)
        layout.add_widget(input_field)
        buttons_layout = BoxLayout(size_hint_y=None, height=50, spacing=10)
        ok_button = Button(text='OK')
        cancel_button = Button(text='Cancel')

        def on_ok(instance):
            if input_field.text.strip() == "":
                self.show_error("Erro", "Entrada não pode ser vazia.")  # Corrigir aqui
                return  # Não fecha o popup e não restaura a tela anterior
            callback(input_field.text, *args)
            popup.dismiss()

        def on_cancel(instance):
            popup.dismiss()
            self.restore_previous_screen()  # Restaura a tela anterior

        ok_button.bind(on_press=on_ok)
        cancel_button.bind(on_press=on_cancel)
        buttons_layout.add_widget(ok_button)
        buttons_layout.add_widget(cancel_button)
        layout.add_widget(buttons_layout)
        popup = Popup(title=title, content=layout, size_hint=(None, None), size=(400, 200))
        popup.open()

    def show_info(self, title, message):
        popup = Popup(title=title, content=Label(text=message), size_hint=(None, None), size=(400, 200))
        popup.open()

    def show_error(self, title, message):
        layout = BoxLayout(orientation='vertical', padding=10)
        label = Label(text=message, size_hint_y=None, height=30)
        layout.add_widget(label)

        popup = Popup(title=title, content=layout, size_hint=(None, None), size=(400, 200))

        def on_dismiss(instance):
            self.restore_previous_screen()  # Restaura a tela anterior ao fechar o popup

        popup.bind(on_dismiss=on_dismiss)  # Conecta o evento de fechar o popup à função de restauração
        popup.open()

        Clock.schedule_once(lambda dt: popup.dismiss(), 3)  # Fecha o popup após 3 segundos

    def show_yesno_dialog(self, title, message, callback, *args):
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text=message, size_hint_y=None, height=30)
        layout.add_widget(label)

        buttons_layout = BoxLayout(size_hint_y=None, height=50, spacing=10)
        yes_button = Button(text='Sim')
        no_button = Button(text='Não')

        def on_yes(instance):
            callback(True, *args)
            popup.dismiss()

        def on_no(instance):
            callback(False, *args)
            popup.dismiss()

        yes_button.bind(on_press=on_yes)
        no_button.bind(on_press=on_no)
        buttons_layout.add_widget(yes_button)
        buttons_layout.add_widget(no_button)
        layout.add_widget(buttons_layout)

        popup = Popup(title=title, content=layout, size_hint=(None, None), size=(400, 200))
        popup.open()

    def save_previous_screen(self):
        self.previous_screen = [widget for widget in self.root.children]  # Salva a tela atual

    def restore_previous_screen(self):
        if self.previous_screen:
            self.root.clear_widgets()
            for widget in reversed(self.previous_screen):  # Restaura a tela anterior
                self.root.add_widget(widget)

    def normalizar_entrada(self, entrada):
        entrada = unidecode(entrada)  # Remove acentos
        entrada = entrada.replace(" ", "").upper()  # Remove espaços e converte para maiúsculas
        if entrada in ["PAPELAO", "PAPELA"]:
            entrada = "PAPELAO"
        if entrada.endswith("S"):
            entrada = entrada[:-1]
        return entrada

    def salvar_orcamento(self, material, processo, numero_logos, cores, local, quantidade, custo_producao, unitario,
                         altura=None, comprimento=None):
        orcamento = {
            "Material": material,
            "Processo": processo,
            "Local": local,
            "Quantidade": quantidade,
            "Custo de Produção": custo_producao,
            "Valor Unitário": unitario,
        }
        if processo.lower() == 'digital':
            orcamento["Altura"] = altura
            orcamento["Comprimento"] = comprimento
        else:
            orcamento["Número de Logos"] = numero_logos
            orcamento["Cores"] = cores

        self.orcamentos.append(orcamento)
        print("Orçamento salvo:", orcamento)  # Debug para verificar o que está sendo salvo

    def mostrar_relatorio(self, material, processo, numero_logos, cores, local, quantidade, custo_producao, unitario):
        self.is_final_report = False
        self.root.clear_widgets()

        relatorio = (
            f"Material: {material}\n"
            f"Processo: {processo}\n"
        )

        # Adiciona condicional para omitir logos e cores se o processo for Digital
        if processo.lower() != "digital":
            relatorio += (
                f"Número de Logos: {numero_logos}\n"
                f"Cores: {cores}\n"
            )

        relatorio += (
            f"Local: {local}\n"
            f"Quantidade: {quantidade}\n"
            f"Custo de Produção: R${custo_producao:.2f}\n"
            f"Valor Unitário: R${unitario:.2f}\n"
        )

        print("Relatório de Orçamento:", relatorio)  # Debug para verificar os dados que serão exibidos

        layout = BoxLayout(orientation='vertical', padding=10)
        label_relatorio = Label(text=relatorio, size_hint_y=None, height=200)
        layout.add_widget(label_relatorio)

        finalizar_button = Button(text='Finalizar Pedido', size_hint_y=None, height=50)
        finalizar_button.bind(on_press=self.perguntar_novo_pedido)
        layout.add_widget(finalizar_button)

        self.root.add_widget(layout)

    def perguntar_novo_pedido(self, instance):
        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text="Gostaria de fazer um novo pedido?", size_hint_y=None, height=30)
        layout.add_widget(label)

        buttons_layout = BoxLayout(size_hint_y=None, height=50, spacing=10)
        sim_button = Button(text='Sim')
        nao_button = Button(text='Não')

        # Cria o popup e armazena a referência
        popup = Popup(title='Novo Pedido', content=layout, size_hint=(None, None), size=(400, 200))

        sim_button.bind(on_press=lambda instance: self.novo_pedido(popup))
        nao_button.bind(on_press=lambda instance: self.mostrar_relatorio_final(popup))

        buttons_layout.add_widget(sim_button)
        buttons_layout.add_widget(nao_button)
        layout.add_widget(buttons_layout)

        popup.open()

    def novo_pedido(self, popup):
        self.root.clear_widgets()
        self.create_main_frame()
        popup.dismiss()  # Fecha o popup

    def mostrar_relatorio_final(self, popup=None):
        self.is_final_report = True  # Este é o relatório final
        self.root.clear_widgets()

        layout = BoxLayout(orientation='vertical', padding=10, spacing=10)
        label = Label(text="Relatório Final dos Pedidos Realizados", size_hint_y=None, height=30)
        layout.add_widget(label)

        relatorio_html = self.gerar_relatorio_html()
        label_relatorio = Label(text=relatorio_html, size_hint_y=None, height=200)
        layout.add_widget(label_relatorio)

        buttons_layout = BoxLayout(size_hint_y=None, height=50, spacing=10)

        finalizar_button = Button(text='Finalizar Programa', size_hint_y=None, height=50)
        finalizar_button.bind(on_press=self.finalizar_programa)
        buttons_layout.add_widget(finalizar_button)
        layout.add_widget(buttons_layout)

        self.root.add_widget(layout)

        if popup:
            popup.dismiss()  # Fecha o popup, se existir

    def finalizar_pedido(self, instance):
        if self.orcamentos:
            # Salvar o relatório em um arquivo
            pasta_relatorios = 'relatorios_salvos'
            if not os.path.exists(pasta_relatorios):
                os.makedirs(pasta_relatorios)
            arquivo_relatorio = os.path.join(pasta_relatorios, 'relatorio_orcamentos.json')
            with open(arquivo_relatorio, 'w') as f:
                json.dump(self.orcamentos, f)

            # Enviar o relatório por email
            self.enviar_email(arquivo_relatorio)

        self.show_info("Pedido Finalizado", "O pedido foi finalizado e o relatório foi enviado por email.")
        self.novo_pedido(None)  # Retorna à tela principal para um novo pedido

    def salvar_relatorio(self, instance):
        # Define o caminho da pasta onde os relatórios serão salvos
        pasta_relatorios = 'relatorios_salvos'

        # Cria a pasta se ela não existir
        if not os.path.exists(pasta_relatorios):
            os.makedirs(pasta_relatorios)

        # Define o nome do arquivo
        arquivo_relatorio = os.path.join(pasta_relatorios, 'relatorio_orcamentos.json')

        # Salva os dados no arquivo
        with open(arquivo_relatorio, 'w') as f:
            json.dump(self.orcamentos, f)

        self.show_info("Salvo", f"Relatório salvo com sucesso na pasta '{pasta_relatorios}'!")

        # Envia o relatório por email após salvar
        self.enviar_email(arquivo_relatorio)

    def enviar_email(self, arquivo_relatorio):
        logging.basicConfig(level=logging.DEBUG)

        # Dados do servidor de email
        smtp_server = 'smtp.gmail.com'
        smtp_port = 587
        smtp_user = 'appgrangrafico@gmail.com'  # Substitua pelo seu endereço de e-mail
        smtp_password = 'z n r h x g e u k i q z c j b g'  # Substitua pela senha de app gerada

        # Dados do email para o usuário
        destinatario = self.users_data[self.current_user]['email']
        copia = 'app.grangrafico@gmail.com'  # Email para cópia
        assunto = 'Relatório de Orçamentos'
        corpo = f"Olá {self.current_user},\n\nSegue em anexo o relatório de orçamentos.\n\nObrigado,\nEquipe Grangrafico"

        # Criação do relatório em HTML
        relatorio_html = self.gerar_relatorio_html()

        # Criação da mensagem para o usuário
        msg = MIMEMultipart()
        msg['From'] = smtp_user
        msg['To'] = destinatario
        msg['Cc'] = copia
        msg['Subject'] = assunto
        msg.attach(MIMEText(corpo, 'plain'))
        msg.attach(MIMEText(relatorio_html, 'html'))  # Assegura que o HTML seja anexado corretamente

        # Anexando o arquivo
        try:
            with open(arquivo_relatorio, 'rb') as attachment:
                part = MIMEBase('application', 'octet-stream')
                part.set_payload(attachment.read())
                encoders.encode_base64(part)
                part.add_header('Content-Disposition', f'attachment; filename= {os.path.basename(arquivo_relatorio)}')
                msg.attach(part)
            logging.debug("Arquivo anexado com sucesso.")
        except Exception as e:
            logging.error(f"Erro ao anexar o arquivo: {e}")
            self.show_error("Erro de Anexação", f"Falha ao anexar o arquivo: {str(e)}")
            return

        # Enviando o email para o usuário
        try:
            logging.debug("Conectando ao servidor SMTP...")
            server = smtplib.SMTP(smtp_server, smtp_port)
            server.starttls()
            logging.debug("Iniciando TLS...")
            server.login(smtp_user, smtp_password)
            logging.debug("Logado com sucesso.")
            server.send_message(msg)
            server.quit()
            logging.debug("Email enviado com sucesso.")
            self.show_info("Email Enviado", f"Relatório enviado para {destinatario} com sucesso!")
        except Exception as e:
            logging.error(f"Erro ao enviar o email: {e}")
            self.show_error("Erro de Email", f"Falha ao enviar o email: {str(e)}")

        # Enviando o email para o app.grangrafico@gmail.com apenas se o processo digital foi selecionado
        if any(orcamento['Processo'].lower() == 'digital' for orcamento in self.orcamentos):
            # Dados do email para o app.grangrafico@gmail.com
            destinatario_app = 'app.grangrafico@gmail.com'
            assunto_app = 'Informações de Impressão Digital'
            corpo_app = (
                f"Detalhes de Impressão Digital acumulados:\n\n"
                f"{self.digital_print_details}\n\n"
                f"Informações do usuário:\n"
                f"Usuário: {self.current_user}\n"
                f"Email: {destinatario}\n"
            )

            # Criação da mensagem para o app.grangrafico@gmail.com
            msg_app = MIMEMultipart()
            msg_app['From'] = smtp_user
            msg_app['To'] = destinatario_app
            msg_app['Subject'] = assunto_app
            msg_app.attach(MIMEText(corpo_app, 'plain'))

            # Enviando o email para o app.grangrafico@gmail.com
            try:
                logging.debug("Conectando ao servidor SMTP para envio ao app.grangrafico@gmail.com...")
                server = smtplib.SMTP(smtp_server, smtp_port)
                server.starttls()
                logging.debug("Iniciando TLS...")
                server.login(smtp_user, smtp_password)
                logging.debug("Logado com sucesso.")
                server.send_message(msg_app)
                server.quit()
                logging.debug("Email enviado com sucesso para app.grangrafico@gmail.com.")
            except Exception as e:
                logging.error(f"Erro ao enviar o email: {e}")
                self.show_error("Erro de Email", f"Falha ao enviar o email para app.grangrafico@gmail.com: {str(e)}")

    def enviar_email_folhas(self, folhas, total_de_logos, valor_folha):
        logging.basicConfig(level=logging.DEBUG)

        # Dados do servidor de email
        smtp_server = 'smtp.gmail.com'
        smtp_port = 587
        smtp_user = 'appgrangrafico@gmail.com'  # Substitua pelo seu endereço de e-mail
        smtp_password = 'z n r h x g e u k i q z c j b g'  # Substitua pela senha de app gerada

        # Dados do email
        destinatario = 'appgrangrafico@gmail.com'
        assunto = 'Informações de Impressão Digital'
        corpo = (f"Detalhes de Impressão Digital:\n\n"
                 f"Folhas necessárias: {folhas}\n"
                 f"Total de logos por folha: {int(total_de_logos)}\n"
                 f"Valor das folhas: R${valor_folha:.2f}")

        # Criação da mensagem
        msg = MIMEMultipart()
        msg['From'] = smtp_user
        msg['To'] = destinatario
        msg['Subject'] = assunto
        msg.attach(MIMEText(corpo, 'plain'))

        # Enviando o email
        try:
            logging.debug("Conectando ao servidor SMTP...")
            server = smtplib.SMTP(smtp_server, smtp_port)
            server.starttls()
            logging.debug("Iniciando TLS...")
            server.login(smtp_user, smtp_password)
            logging.debug("Logado com sucesso.")
            server.send_message(msg)
            server.quit()
            logging.debug("Email enviado com sucesso.")
        except Exception as e:
            logging.error(f"Erro ao enviar o email: {e}")
            self.show_error("Erro de Email", f"Falha ao enviar o email: {str(e)}")

    def gerar_relatorio_html(self):
        html_template = """
        <html>
        <head>
            <style>
                table {{ width: 100%; border-collapse: collapse; }}
                th, td {{ border: 1px solid black; padding: 8px; text-align: left; }}
                th {{ background-color: #f2f2f2; }}
            </style>
        </head>
        <body>
            <h2>Relatório de Orçamentos</h2>
            <table>
                <tr>
                    <th>Material</th>
                    <th>Processo</th>
                    <th>{col1}</th>
                    <th>{col2}</th>
                    <th>Local</th>
                    <th>Quantidade</th>
                    <th>Custo de Produção</th>
                    <th>Valor Unitário</th>
                </tr>
        """  # 1

        rows = ""
        col1 = "Número de Logos"  # 2
        col2 = "Cores"  # 2
        for orcamento in self.orcamentos:
            print(f"Processando orçamento: {orcamento}")  # Debug print para verificar o conteúdo de cada orçamento
            if orcamento['Processo'].lower() == 'digital':
                col1 = "Altura do Logo (mm)"  # 3
                col2 = "Comprimento do Logo (mm)"  # 3
                row = f"""
                    <tr>
                        <td>{orcamento['Material']}</td>
                        <td>{orcamento['Processo']}</td>
                        <td>{orcamento.get('Altura', 'N/A')}</td>
                        <td>{orcamento.get('Comprimento', 'N/A')}</td>
                        <td>{orcamento.get('Local', 'N/A')}</td>
                        <td>{orcamento['Quantidade']}</td>
                        <td>R${orcamento['Custo de Produção']:.2f}</td>
                        <td>R${orcamento['Valor Unitário']:.2f}</td>
                    </tr>
                """  # 4
            else:
                row = f"""
                    <tr>
                        <td>{orcamento['Material']}</td>
                        <td>{orcamento['Processo']}</td>
                        <td>{orcamento.get('Número de Logos', 'N/A')}</td>
                        <td>{orcamento.get('Cores', 'N/A')}</td>
                        <td>{orcamento.get('Local', 'N/A')}</td>
                        <td>{orcamento['Quantidade']}</td>
                        <td>R${orcamento['Custo de Produção']:.2f}</td>
                        <td>R${orcamento['Valor Unitário']:.2f}</td>
                    </tr>
                """  # 5
            rows += row

        html = html_template + rows + """
            </table>
        </body>
        </html>
        """

        print(f"col1: {col1}, col2: {col2}")  # Debug print para verificar col1 e col2
        formatted_html = html.format(col1=col1, col2=col2)  # 6
        print(f"Formatted HTML: {formatted_html}")  # Debug print para verificar HTML formatado
        return formatted_html

    def finalizar_programa(self, instance):
        if self.orcamentos:
            # Salvar o relatório em um arquivo
            pasta_relatorios = 'relatorios_salvos'
            if not os.path.exists(pasta_relatorios):
                os.makedirs(pasta_relatorios)
            arquivo_relatorio = os.path.join(pasta_relatorios, 'relatorio_orcamentos.json')
            with open(arquivo_relatorio, 'w') as f:
                json.dump(self.orcamentos, f)
            # Enviar o relatório por email
            self.enviar_email(arquivo_relatorio)
        App.get_running_app().stop()

    def escolher_produto(self, produto):
        self.root.clear_widgets()
        if produto == 'CANETAS - LAPIS':
            self.escolher_pecas_canetas()
        elif produto == 'BOLSAS - SACOLAS':
            self.escolher_processo_bolsa()
        elif produto == 'NECESSERIES - MOCHILAS':
            self.escolher_pecas_necesseries()
        elif produto == 'SQUEZE - COPOS - GARRAFAS':
            self.escolher_pecas_squize()
        elif produto == "AGENDAS - BLOCOS - PORTA JOIAS":
            self.escolher_pecas_agendas()
        elif produto == "CHAVEIRO - PLAQUINHAS":
            self.escolher_processo_chaveiro()
        elif produto == "NENHUMA DAS ALTERNATIVAS":
            self.contato_vendedor()
        else:
            self.show_info("Info", f"Funcionalidade para {produto} não implementada.")

    def escolher_pecas_canetas(self):
        self.show_input_dialog("Escolha Caneta ou Lápis", "Qual tipo de produto? (CANETA ou LAPIS)",
                               self.escolher_pecas)

    def escolher_pecas(self, peca):
        peca = self.normalizar_entrada(peca)
        if peca == "CANETA":
            self.escolher_material()
        elif peca == "LAPI":
            self.escolher_processo_lapis()
        else:
            self.show_error("Erro", "Peça inválida. Por favor, escolha uma das opções fornecidas.")

    def escolher_material(self):
        self.show_input_dialog("Escolha o Material", "Tipo de Material? (Plástico, Papelão, Bambu, Metal)",
                               self.processar_material,)

    def processar_material(self, material):
        material = self.normalizar_entrada(material)
        if material in ["PLASTICO", "PAPELAO", "BAMBU", "METAL"]:
            caneta = Caneta(self, material)
            caneta.escolher_processo()
        else:
            self.show_error("Erro", "Material inválido. Por favor, escolha uma das opções fornecidas.")

    def escolher_processo_chaveiro(self):
        self.show_input_dialog("Escolha o Material", "Tipo de Material? (Plástico, Papelão, Bambu, Metal)",
                               self.processar_chaveiro )

    def processar_chaveiro(self,material):
        material = self.normalizar_entrada(material)
        if material in ["PLASTICO", "PAPELAO", "BAMBU", "METAL"]:
            chaveiro = Chaveiro(self, material)
            chaveiro.escolher_processo()
        else:
            self.show_error("Erro", "Material inválido. Por favor, escolha uma das opções fornecidas.")

    def escolher_processo_lapis(self):
        lapis = Lapis(self)
        lapis.escolher_processo()

    def escolher_processo_bolsa(self):
        bolsa = Bolsa(self)
        bolsa.escolher_processo()

    def escolher_pecas_necesseries(self):
        necesseries = Necesseries(self)
        necesseries.escolher_processo()

    def escolher_pecas_squize(self):
        squize = Squize(self)
        squize.escolher_processo()

    def escolher_pecas_agendas(self):
        agenda = Agenda(self)
        agenda.escolher_processo()



    def contato_vendedor(self):
        self.show_input_dialog("Contato",
                               "Por favor, entre em contato com o vendedor para um orçamento mais detalhado. Gostaria de retornar ao aplicativo?",
                               self.retorno_contato_vendedor)

    def retorno_contato_vendedor(self, resposta):
        if resposta.lower() == 'sim':
            self.root.clear_widgets()
            self.create_main_frame()
        else:
            self.root.quit()

    def print_user_data(self):
        if os.path.exists(self.user_data_file):
            with open(self.user_data_file, 'r') as f:
                data = json.load(f)
                print(json.dumps(data, indent=4))
        else:
            print("O arquivo user_data.json não existe.")

    def processar_processo_digital(self, quantidade, alt, comp, processo="Digital"):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        custo_producao = self.calcular_custo_producao_digital(alt, comp, quantidade)
        unitario = custo_producao / quantidade if quantidade != 0 else 0

        self.salvar_orcamento("NECESSERIES", processo, None, None, "Digital", quantidade, custo_producao, unitario,
                              altura=alt, comprimento=comp)
        self.mostrar_relatorio("NECESSERIES", processo, None, None, "Digital", quantidade, custo_producao, unitario)

    def calcular_custo_producao_digital(self, alt, comp, quantidade):
        entrada_minima_dtf = 50  # Exemplo de valor, ajuste conforme necessário
        self.show_info("Produção Digital", "Lembre-se que para produção digital, o seu produto terá que ser analisado!")

        medidas_alt = alt + (alt * 33.33 / 100)  # soma da medida do logo mais espaço entre logo
        logo_vertical = 400 / medidas_alt  # quantidade de logos na vertical para uma folha

        medidas_comp = comp + (comp * 33.33 / 100)  # soma da medida do logo na horizontal
        logo_horizontal = 300 / medidas_comp

        total_de_logos = logo_horizontal * logo_vertical  # total de logos em uma folha
        folhas = max(1, (quantidade + total_de_logos - 1) // total_de_logos)

        valor_folha = (folhas * 20) * 2

        custo_logo_unidade = 1.5
        custo_producao = (quantidade * custo_logo_unidade) + entrada_minima_dtf + valor_folha

        custo_producao = round(custo_producao, 2)

        # Exibir informações sobre folhas e logos
        self.show_info("Detalhes de Impressão Digital",
                       f"Folhas necessárias: {folhas}\nTotal de logos por folha: {int(total_de_logos)}")

        # Acumular detalhes de impressão digital
        self.digital_print_details += (
            f"\nDetalhes de Impressão Digital:\n"
            f"Folhas necessárias: {folhas}\n"
            f"Total de logos por folha: {int(total_de_logos)}\n"
            f"Valor das folhas: R${valor_folha:.2f}\n"
        )

        return custo_producao

    def processar_quantidade_digital(self, quantidade, alt, comp, processo="Digital"):
        try:
            quantidade = int(quantidade)
        except ValueError:
            self.show_error("Erro", "Por favor, insira um número válido para a quantidade.")
            return

        custo_producao = self.calcular_custo_producao_digital(alt, comp, quantidade)
        unitario = custo_producao / quantidade if quantidade != 0 else 0

        self.salvar_orcamento("NECESSERIES", processo, None, None, "Digital", quantidade, custo_producao, unitario,
                              altura=alt, comprimento=comp)
        self.mostrar_relatorio("NECESSERIES", processo, None, None, "Digital", quantidade, custo_producao, unitario)


if __name__ == "__main__":
    AppOrcamento().run()

