---
title: "Concourse: Como buildar imagem multiplataforma"
date: 2024-09-11T14:20:52-03:00
description: ""
draft: false
tags: ["concourse", "multiplatform", "image", "docker", "buildx", "ci", "oci-build-task"]
math: false
toc: true
---

Como criar uma imagem multiplataforma utilizando o [Concourse CI](https://concourse-ci.org)? Esse artigo irá mostrar como.

Com a crescente adoção de arquiteturas variadas, como ARM para dispositivos móveis e cloud computing, a criação de imagens Docker que suportem múltiplas plataformas se tornou essencial para manter a eficiência no ciclo de desenvolvimento. Imagens multiplataforma simplificam a integração contínua, eliminando a necessidade de manter builds separados para cada arquitetura.

Para gerar uma imagem docker seja multiplataforma ou não, é utilizado o buildkit [^1] em que é um mecanismo de construção de imagens do [docker](https://www.docker.com).

O buildx [^2] é uma extensão do Docker CLI que habilita o uso das funcionalidades avançadas do BuildKit de maneira mais acessível e amigável para o usuário. Com Buildx, você pode:

- Build Multiplataforma: Criar imagens que funcionam em várias arquiteturas (como amd64 e arm64) a partir de uma única plataforma.
- Múltiplos Drivers: Buildx permite usar diferentes drivers de build, como docker, kubernetes, e qemu, para construir imagens em ambientes variados.
- Fácil Configuração: Configurar builds multiplataforma se torna muito mais fácil e direto com o Buildx, usando comandos simples do Docker CLI

## Concourse

Para o build das imagens utilizamos o componente [oci-build-task](https://github.com/concourse/oci-build-task) na versão >= [v0.11.0](https://github.com/concourse/oci-build-task/tree/v0.11.0).

Na task de build devemos passar dois novos parâmetros, são eles: `IMAGE_PLATFORM` e `OUTPUT_OCI`. O parâmetro de `IMAGE_PLATFORM` passamos uma lista de arquiteturas separadas com vírgula. Já o parâmetro `OUTPUT_OCI` é um boolean e é obrigatório passarmos como true para o build multiplataforma. Abaixo deixo o código de exemplo:

```yaml
- task: build-image
  privileged: true
  config:
    platform: linux
    image_resource:
      type: registry-image
      source:
        repository: concourse/oci-build-task
        tag: 0.11.1
    inputs:
    - name: dockerfile
    outputs:
    - name: image
    params:
      CONTEXT: dockerfile
      IMAGE_PLATFORM: linux/amd64,linux/arm64
      OUTPUT_OCI: true
    run:
      path: build
```

Desta forma será gerado uma imagem do tipo [OCI](https://opencontainers.org), para as arquiteturas passadas no parâmetro `IMAGE_PLATFORM`.

Para realizar o push da imagem podemos realizar de duas formas no concourse: put ou utilizando o componente [registry-image-resource](https://github.com/concourse/registry-image-resource). Utilizando o put é algo simples e adicionamos apenas a task de put:

```yaml
- put: multi-platform-docker-image
  params:
    image: image/image
```

O parâmetro image é importante constar o path absoluto da imagem gerada, sem a extensão `.tar`[^3] [^4].

Caso necessite realizar o push da imagem utilizando o script de out[^5] do registry-image-registry, basta utilizarmos a versão >= X.X.X e passar o json:

```json
{
    "repository": "registry.example.com",
    "username": "<user>",
    "password": "<pass>",
    "image": "image/image"
}
```

## Referências

[^1]: https://github.com/docker/buildx?tab=readme-ov-file#building-multi-platform-images
[^2]: https://docs.docker.com/build/building/multi-platform/
[^3]: https://github.com/concourse/oci-build-task/issues/85#issuecomment-2092396915
[^4]: https://github.com/concourse/oci-build-task/pull/93/files#diff-b335630551682c19a781afebcf4d07bf978fb1f8ac04c6bf87428ed5106870f5
[^5]: https://github.com/concourse/registry-image-resource?tab=readme-ov-file#put-step-out-script-push-and-tag-an-image
