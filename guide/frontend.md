# Guía Maestra: Desarrollo Angular con API Externa

## RECORDATORIO MAC: Usa sudo antes de comandos globales (sudo npm install -g...) y si tienes problemas de permisos al crear archivos, usa sudo chown -R $USER . en la terminal dentro de tu carpeta.

# Fase 1

### 1) Crear proyecto 
```bash
ng new nombre-del-proyecto
```

### 2) Instalacion de librerias
Ejecutar estos comando en la carpeta del proyecto
### UI de Material
```bash
ng add @angular/material 
```

### Internacionalización (Traducción)
```bash
npm install @ngx-translate/core @ngx-translate/http-loader --save
```

### Parche de Reactividad (Prevenir pantalla blanca)
```bash
npm install zone.js --save
```

# Fase 2

Para que el proyecto entienda las llamadas HTTP y las traducciones, debes configurar los proveedores principales.
En **app.config.ts**
```bash
import { ApplicationConfig, importProvidersFrom } from '@angular/core';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { HttpClient, provideHttpClient } from '@angular/common/http';
import { TranslateLoader, TranslateModule } from '@ngx-translate/core';
import { TranslateHttpLoader } from '@ngx-translate/http-loader';

export function HttpLoaderFactory(http: HttpClient) {
  return new TranslateHttpLoader(http, './assets/i18n/', '.json');
}

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes), 
    provideAnimationsAsync(),
    provideHttpClient(),
    importProvidersFrom(
      TranslateModule.forRoot({
        loader: {
          provide: TranslateLoader,
          useFactory: HttpLoaderFactory,
          deps: [HttpClient]
        },
        defaultLanguage: 'es' // Idioma por defecto
      })
    )
  ]
};
```

En **main.ts**
```bash
import 'zone.js'; // <- ESTO DEBE IR PRIMERO SIEMPRE
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { App } from './app/app'; // Asegúrate de que la ruta coincida con tu componente raíz

bootstrapApplication(App, appConfig)
  .catch((err) => console.error(err));
```

# Fase 3

1. Crea la ruta de carpetas: public/assets/i18n/
2. Crea dos archivos dentro: es.json y en.json.

En es.json
```bash
{
  "toolbar": {
    "title": "Mi Aplicación"
  },
  "content": {
    "title": "Lista de Recursos",
    "id": "Código",
    "name": "Nombre",
    "description": "Descripción",
    "action": "Ver Detalle"
  }
}
```

En en.json
```bash
{
  "toolbar": {
    "title": "My Application"
  },
  "content": {
    "title": "Resource List",
    "id": "Code",
    "name": "Name",
    "description": "Description",
    "action": "View Detail"
  }
}
```

# Fase 4

Comando para generar la carpeta
```bash
ng generate environments
```

En src/environments/environment.development.ts:
```bash
export const environment = {
  production: false,
  serverBasePath: 'https://api.dominio-del-profesor.com', // <- REEMPLAZAR CON API REAL
  resourceEndpoint: '/api/v1/recursos' // <- REEMPLAZAR CON ENDPOINT
};
```

# Fase 5
Imaginemos que el recurso del examen se llama item. Ejecuta estos comandos para crear la arquitectura DDD completa:
```bash
# Capa Domain
ng generate class item/domain/model/item --type=entity --skip-tests=true

# Capa Infrastructure
ng generate interface item/infrastructure/item-response
ng generate class item/infrastructure/item-assembler --skip-tests=true
ng generate service item/infrastructure/item-api --skip-tests=true

# Capa Application
ng generate service item/application/item-app --skip-tests=true

# Capa Presentation (Componentes)
ng generate component item/presentation/components/item-list --skip-tests=true
ng generate component item/presentation/components/item-card --skip-tests=true

# Componentes Compartidos (Header/Layout)
ng generate component shared/presentation/components/header --skip-tests=true
ng generate component shared/presentation/components/layout --skip-tests=true
```

# Fase 6
Interface item-response.ts
```bash
export interface ItemResponse {
  // Pon exactamente lo que devuelve el JSON de internet
  id: number;
  data_name: string;
  nested_info: {
    picture: string;
    details: string;
  };
}

Entity item.entity.ts
```bash
export class Item {
  id: number;
  name: string;
  description: string;
  imageUrl: string; // Siempre verifica que este nombre coincida luego en el HTML

  constructor() {
    this.id = 0;
    this.name = '';
    this.description = '';
    this.imageUrl = '';
  }
}
```

Assembler item-assembler.ts
```bash
import { ItemResponse } from './item-response';
import { Item } from '../domain/model/item.entity';

export class ItemAssembler {
  static toEntityFromResponseArray(responses: ItemResponse[]): Item[] {
    return responses.map(res => this.toEntityFromResponse(res));
  }

  static toEntityFromResponse(res: ItemResponse): Item {
    return {
      id: res.id,
      name: res.data_name || 'Sin nombre',
      description: res.nested_info?.details || 'Sin descripción',
      imageUrl: res.nested_info?.picture || 'https://via.placeholder.com/150' // Fallback si no hay imagen
    };
  }
}
```

API Service item-api.service.ts
```bash
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '../../../environments/environment.development';
import { map, Observable } from 'rxjs';
import { Item } from '../domain/model/item.entity';
import { ItemResponse } from './item-response';
import { ItemAssembler } from './item-assembler';

@Injectable({ providedIn: 'root' })
export class ItemApi {
  private http = inject(HttpClient);
  private url = `${environment.serverBasePath}${environment.resourceEndpoint}`;

  getItems(): Observable<Item[]> {
    return this.http.get<ItemResponse[]>(this.url)
      .pipe(map(response => ItemAssembler.toEntityFromResponseArray(response)));
  }
}
```

# Fase 7
item-app.service.ts
```bash
import { computed, inject, Injectable, signal } from '@angular/core';
import { Item } from '../domain/model/item.entity';
import { ItemApi } from '../infrastructure/item-api.service';

@Injectable({ providedIn: 'root' })
export class ItemApp {
  private api = inject(ItemApi);
  private itemsSignal = signal<Item[]>([]);
  
  public items = computed(() => this.itemsSignal());

  loadItems(): void {
    if (this.itemsSignal().length === 0) {
      this.api.getItems().subscribe(data => this.itemsSignal.set(data));
    }
  }
}
```

# Fase 8
header.ts
```bash
import { Component, inject } from '@angular/core';
import { MatToolbarModule } from '@angular/material/toolbar';
import { MatButtonModule } from '@angular/material/button';
import { TranslateModule, TranslateService } from '@ngx-translate/core';

@Component({
  selector: 'app-header',
  standalone: true,
  imports: [MatToolbarModule, MatButtonModule, TranslateModule],
  template: `
    <mat-toolbar color="primary">
      <span>{{ 'toolbar.title' | translate }}</span>
      <span style="flex: 1 1 auto;"></span>
      <button mat-button (click)="changeLanguage('es')">ES</button>
      <button mat-button (click)="changeLanguage('en')">EN</button>
    </mat-toolbar>
  `
})
export class Header {
  private translate = inject(TranslateService);
  changeLanguage(lang: string) {
    this.translate.use(lang);
  }
}
```

layout.ts
```bash
import { Component, inject, OnInit } from '@angular/core';
import { Header } from '../header/header';
import { ItemList } from '../../../../item/presentation/components/item-list/item-list';
import { ItemApp } from '../../../../item/application/item-app.service';

@Component({
  selector: 'app-layout',
  standalone: true,
  imports: [Header, ItemList],
  template: `
    <app-header></app-header>
    <main style="padding: 20px;">
      <app-item-list [items]="appService.items()"></app-item-list>
    </main>
  `
})
export class Layout implements OnInit {
  appService = inject(ItemApp);
  ngOnInit() { this.appService.loadItems(); }
}
```

item-list.ts
```bash
import { Component, input } from '@angular/core';
import { Item } from '../../../domain/model/item.entity';
import { ItemCard } from '../item-card/item-card';
import { MatGridListModule } from '@angular/material/grid-list';
import { TranslateModule } from '@ngx-translate/core';

@Component({
  selector: 'app-item-list',
  standalone: true,
  imports: [ItemCard, MatGridListModule, TranslateModule],
  template: `
    <h1 style="text-align: center;">{{ 'content.title' | translate }}</h1>
    <mat-grid-list cols="3" rowHeight="400px" gutterSize="15px">
      @for (element of items(); track element.id) {
        <mat-grid-tile>
          <app-item-card [item]="element"></app-item-card>
        </mat-grid-tile>
      }
    </mat-grid-list>
  `
})
export class ItemList {
  items = input.required<Item[]>();
}
```

item-card.ts
```bash
import { Component, input } from '@angular/core';
import { Item } from '../../../domain/model/item.entity';
import { MatCardModule } from '@angular/material/card';
import { MatButtonModule } from '@angular/material/button';
import { TranslateModule } from '@ngx-translate/core';

@Component({
  selector: 'app-item-card',
  standalone: true,
  imports: [MatCardModule, MatButtonModule, TranslateModule],
  template: `
    <mat-card appearance="outlined" style="width: 100%; height: 100%;">
      <img mat-card-image [src]="item().imageUrl" [alt]="item().name" style="height: 200px; object-fit: cover;">
      
      <mat-card-content>
        <h2 style="margin-top: 10px;">{{ item().name }}</h2>
        <p><strong>{{ 'content.id' | translate }}:</strong> {{ item().id }}</p>
        <p><strong>{{ 'content.description' | translate }}:</strong> {{ item().description }}</p>
      </mat-card-content>

      <mat-card-actions>
        <button mat-flat-button color="primary" style="width: 100%;">
          {{ 'content.action' | translate }}
        </button>
      </mat-card-actions>
    </mat-card>
  `
})
export class ItemCard {
  item = input.required<Item>();
}
```

# Fase 9 
Por último, enruta todo abriendo tu archivo src/app/app.ts (o app.component.ts) y el html principal.

app.ts
```bash
import { Component } from '@angular/core';
import { Layout } from './shared/presentation/components/layout/layout';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [Layout],
  templateUrl: './app.html'
})
export class App {}
```

app.html
<app-layout></app-layout>

