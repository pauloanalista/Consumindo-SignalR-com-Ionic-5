# Consumindo SignalR com Ionic 5
 Veja um exemplo de uma aplicação Ionic 5 consumindo um serviço SignalR

Estarei disponibilizando os arquivos para facilitar a vida de quem deseja implementar algo depois.

## Serviço SignalR
Serviço SignalR levantado por uma API em ASPNET.Core

## Front-End
Nosso front-end iremos usar uma aplicação Ionic 5

### Configurando nossa API
Crie uma classe em sua API que será responsável pelo HUB do SignalR.
Hub é basicamente um canal onde as aplicações devem se conectar.
```csharp
public class ChatHub : Hub
    {
        public async Task AtualizarChat(Guid idChat, EnumStatus status, Guid idRespostaChat, DateTime data, Guid idUsuario, string nomeUsuario,  Guid idCliente, string nomeCliente, string mensagem)
        {
            await Clients.All.SendAsync("ObterMensagemChat", idChat, status, idRespostaChat, data, idUsuario, nomeUsuario, idCliente, nomeCliente, mensagem);
        }

        public async Task AvisarStatusChat(Guid idChat, EnumStatus status)
        {
            await Clients.All.SendAsync("OuvirStatusChat", idChat, status);
        }

        public async Task AvisarDigitando(Guid idChat, bool isCliente)
        {
            await Clients.All.SendAsync("OuvirDigitando", idChat, isCliente);
        }
    }

```
É necessário configurar sua API para usar o SignalR, siga o exemplo abaixo.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http.Features;
using Microsoft.AspNetCore.Internal;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;


namespace Qsti.Suporte.Api
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.Configure<FormOptions>(options =>
            {
                options.ValueLengthLimit = int.MaxValue; //not recommended value
                options.MultipartBodyLengthLimit = long.MaxValue; //not recommended value
            });

            services.ConfigureMediatR();
            services.ConfigureServices();
            services.ConfigureRepositories();
            services.ConfigureSwagger();
            services.ConfigureAuthentication();
            services.ConfigureMVC();
            services.AddHttpContextAccessor();


            services.Configure<IISOptions>(options =>
            {
                options.AutomaticAuthentication = false;
            });

            services.AddSignalR(); //CONFIGURAÇÃO SIGNALR
            
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseCors(x =>
            {
                x.AllowAnyOrigin();
                x.AllowAnyHeader();
                x.AllowAnyMethod();
                x.AllowCredentials();
                x.Build();
            });

            //Configura para usarmos o MVC
            app.UseMvc();

            //CONFIGURAÇÃO SIGNALR
            app.UseSignalR(routes =>
            {
                routes.MapHub<ChatHub>("/hub");
            });

            //Cria a documentação da Api de forma automatica
            app.UseSwagger();
            app.UseSwaggerUI(c => {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "Suporte - V1");
            });

            
        }
    }

    
}

```
O serviço ficara disponível em {seu dominio}/hub. ex: meusite.com.br/hub

### Conectando o Ionic 5 no SignalR
Abaixo eu separo os trechos de código utilizado para consumir o SignalR através do Ionic 5.

Vamos criar um service para poder conectar com o SignalR chamado SignalRService.
Não se esqueça de instalar o pacote nuget "@aspnet/signalr"
```sh
npm i @aspnet/signalr
```
```typescript
import { Injectable } from '@angular/core';
import { LoadingController, AlertController, ToastController } from '@ionic/angular';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Subject } from 'rxjs';
import * as signalR from "@aspnet/signalr";
import { UtilService } from './util.service';

@Injectable({
  providedIn: 'root'
})
export class SignalRService {
  
  private hubConnection: signalR.HubConnection
  constructor(private utilService : UtilService) { }

  //Inicia a conexão com o SignalR, no console será apresentado se conectou ou deu erro
  public startConnection = () => {
    this.hubConnection = new signalR.HubConnectionBuilder()
                            .withUrl(this.utilService.obterUrlDoSignalR(),{  //this.utilService.obterUrlDoSignalR() retorna onde esta o Hub {dominio do site}/hub
                              skipNegotiation: true,
                              transport: signalR.HttpTransportType.WebSockets
                            })
                            .build();
    this.hubConnection
      .start()
      .then(() => console.log('Conectado no SignalR'))
      .catch(err => console.log('Erro ao tentar conectar no SignalR: ' + err))
  }
  
  //Retorna a conexão do SignalR
  getHubConnection() : signalR.HubConnection{
    return this.hubConnection;
  }
}

```

Consumindo os métodos do SignalR em uma página qualquer
```typescript
import { Component, OnInit } from '@angular/core';
import { ChatService } from 'src/app/services/chat.service';
import { ActivatedRoute } from '@angular/router';
import { AlertController, NavController } from '@ionic/angular';
import { UtilService } from 'src/app/services/util.service';
import { SignalRService } from 'src/app/services/signalr.service';


@Component({
  selector: 'app-chat-conversar',
  templateUrl: './chat-conversar.page.html',
  styleUrls: ['./chat-conversar.page.scss'],
})
export class ChatConversarPage implements OnInit {

  constructor(public signalRService: SignalRService) {
   
    
  }
  ngOnInit() {
    this.signalRService.startConnection();

    this.signalRService.getHubConnection().on('ObterMensagemChat', (idChat, status, idRespostaChat, data, idUsuario, nomeUsuario, idCliente, nomeCliente, mensagem) => {
      
      console.log(idChat, status, idRespostaChat, data, idUsuario, nomeUsuario, idCliente, nomeCliente, mensagem);
    });

    
    this.signalRService.getHubConnection().on('OuvirStatusChat', (idChat, status) => {
      console.log('MUDOU STATUS CHAT', idChat, status);
    });

    this.signalRService.getHubConnection().on('OuvirDigitando', (idChat, isCliente) => {
      console.log('OuvirDigitando', idChat, isCliente);

    });

  }

  digitando(){
    this.signalRService.getHubConnection().send('AvisarDigitando', this.idChat, false);
  }
}
```
# VEJA TAMBÉM
## Grupo de Estudo no Telegram
- [Participe gratuitamente do grupo de estudo](https://t.me/blogilovecode)

## Cursos baratos!
- [Meus cursos](https://olha.la/udemy)

## Novidades, cupons de descontos e cursos gratuitos
https://olha.la/ilovecode-receber-cupons-novidades


