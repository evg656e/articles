# Вывод типов в jscodeshift и TypeScript

Начиная с версии 6.0 [jscodeshift](https://github.com/facebook/jscodeshift) поддерживает работу с TypeScript (далее TS). В процессе написания codemode-ов (далее, преобразований), может потребоваться узнать тип переменной, которая не имеет явной аннотации. К сожалению, jscodeshift не предоставляет средств для вывода типов «из коробки».

Рассмотрим пример. Допустим, мы хотим написать преобразование, которое добавляет явный тип возвращаемого значения для функций и методов классов. Т.е. имея на входе:

```typescript
function foo(x: number) {
    return x;
}
```

Мы хотим получить на выходе:

```typescript
function foo(x: number): number {
    return x;
}
```

К сожалению, в общем случае, решение такой задачи очень нетривиально. Вот лишь несколько примеров:

```typescript
function toString(x: number) {
    return '' + x;
}

function toInt(str: string) {
    return parseInt(str);
}

function toIntArray(strings: string[]) {
    return strings.map(Number.parseInt);
}

class Foo1 {
    constructor(public x = 0) { }

    getX() {
        return this.x;
    }
}

class Foo2 {
    x: number;

    constructor(x = 0) {
        this.x = x;
    }

    getX() {
        return this.x;
    }
}

function foo1(foo: Foo1) {
    return foo.getX();
}

function foo2(foo: Foo2) {
    return foo.getX();
}
```

К счастью, задача вывода типов уже решена внутри компилятора TS. [API компилятора](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API) предоставляет средства для вывода типов, которые можно использовать для написания преобразования.

Однако, просто взять и воспользоваться компилятором TS, переопределив парсер jscodeshift, нельзя. Дело в том, что jscodeshift ожидает от внешних парсеров абстрактное  синтаксическое дерево (AST) в формате [ESTree](https://github.com/estree/estree). А AST компилятора TS таковым не является.

Конечно, можно было бы воспользоваться компилятором TS и без использования jscodeshift, написав преобразование «с нуля». Либо же воспользоваться одним из средств, которые существуют в комьюнити TS, например, [ts-morph](https://github.com/dsherret/ts-morph). Но для многих jscodeshift будет более привычным и выразительным решением. Поэтому далее будет рассмотрено, как обойти это ограничение.

Идея состоит в том, чтобы получить отображение из AST парсера jscodeshift (далее ESTree) в AST компилятора TS (далее TSTree), и затем воспользоваться средствами вывода типов компилятора TS. Далее будут рассмотрены два способа реализации этой идеи.

## Отображение с использованием номеров строк и столбцов

Первый способ использует номера строк и столбцов (позиции) узлов, чтобы найти отображение из TSTree в ESTree. Несмотря на то, что в общем случае позиции узлов могут не совпадать, почти всегда можно найти нужное отображение в каждом конкретном случае.

Итак, напишем преобразование, которое выполнит задачу добавления явных аннотаций. Напомню, на выходе мы должны получить следующее:

```typescript
function toString(x: number): number {
    return '' + x;
}

function toInt(str: string): number {
    return parseInt(str);
}

function toIntArray(strings: string[]): number[] {
    return strings.map(Number.parseInt);
}

class Foo1 {
    constructor(public x = 0) { }

    getX(): number {
        return this.x;
    }
}

class Foo2 {
    x: number;

    constructor(x = 0) {
        this.x = x;
    }

    getX(): number {
        return this.x;
    }
}

function foo1(foo: Foo1): number {
    return foo.getX();
}

function foo2(foo: Foo2): number {
    return foo.getX();
}
```

Сначала, нам нужно построить TSTree и получить `typeChecker` компилятора TS:

```javascript
const compilerOptions = {
    target: ts.ScriptTarget.Latest
};

const program = ts.createProgram([path], compilerOptions);
const sourceFile = program.getSourceFile(path);
const typeChecker = program.getTypeChecker();
```

Далее, построим отображение из ESTree в TSTree с использованием стартовой позиции. Для этого будем использовать двухуровневый `Map` (первый уровень – для строк, второй уровень – для столбцов, результат – узел TSTree):

```javascript
const locToTSNodeMap = new Map();

const esTreeNodeToTSNode = ({ loc: { start: { line, column } } }) => locToTSNodeMap.has(line) ? locToTSNodeMap.get(line).get(column) : undefined;

(function buildLocToTSNodeMap(node) {
    const { line, character } = sourceFile.getLineAndCharacterOfPosition(node.getStart(sourceFile));
    const nextLine = line + 1;
    if (!locToTSNodeMap.has(nextLine))
        locToTSNodeMap.set(nextLine, new Map());
    locToTSNodeMap.get(nextLine).set(character, node);
    ts.forEachChild(node, buildLocToTSNodeMap);
}(sourceFile));
```

Необходимо скорректировать номер строки, т.к. в TSTree номера строк начинаются с нуля, а в ESTree – с единицы.

Далее нам надо обойти все функции и методы классов, проверить возвращаемый тип и если он равен `null`, добавить аннотацию типа:

```javascript
const ast = j(source);
ast
    .find(j.FunctionDeclaration)
    .forEach(({ value }) => {
        if (value.returnType === null)
            value.returnType = getReturnType(esTreeNodeToTSNode(value));
    });
ast
    .find(j.ClassMethod, { kind: 'method' })
    .forEach(({ value }) => {
        if (value.returnType === null)
            value.returnType = getReturnType(esTreeNodeToTSNode(value).parent);
    });
return ast.toSource();
```

Пришлось скорректировать код для получения узла метода класса, т.к. по стартовой позиции узла метода в ESTree в TSTree находится узел идентификатора метода (поэтому мы используем `parent`-а).

Наконец, напишем код получения аннотации возвращаемого типа:

```javascript
function getReturnTypeFromString(typeString) {
    let ret;
    j(`function foo(): ${typeString} { }`)
        .find(j.FunctionDeclaration)
        .some(({ value: { returnType } }) => ret = returnType);
    return ret;
}

function getReturnType(node) {
    return getReturnTypeFromString(
        typeChecker.typeToString(
            typeChecker.getReturnTypeOfSignature(
                typeChecker.getSignatureFromDeclaration(node)
            )
        )
    );
}
```

Полный листинг:

```javascript
import * as ts from 'typescript';

export default function transform({ source, path }, { j }) {
    const compilerOptions = {
        target: ts.ScriptTarget.Latest
    };

    const program = ts.createProgram([path], compilerOptions);
    const sourceFile = program.getSourceFile(path);
    const typeChecker = program.getTypeChecker();

    const locToTSNodeMap = new Map();

    const esTreeNodeToTSNode = ({ loc: { start: { line, column } } }) => locToTSNodeMap.has(line) ? locToTSNodeMap.get(line).get(column) : undefined;

    (function buildLocToTSNodeMap(node) {
        const { line, character } = sourceFile.getLineAndCharacterOfPosition(node.getStart(sourceFile));
        const nextLine = line + 1;
        if (!locToTSNodeMap.has(nextLine))
            locToTSNodeMap.set(nextLine, new Map());
        locToTSNodeMap.get(nextLine).set(character, node);
        ts.forEachChild(node, buildLocToTSNodeMap);
    }(sourceFile));

    function getReturnTypeFromString(typeString) {
        let ret;
        j(`function foo(): ${typeString} { }`)
            .find(j.FunctionDeclaration)
            .some(({ value: { returnType } }) => ret = returnType);
        return ret;
    }

    function getReturnType(node) {
        return getReturnTypeFromString(
            typeChecker.typeToString(
                typeChecker.getReturnTypeOfSignature(
                    typeChecker.getSignatureFromDeclaration(node)
                )
            )
        );
    }

    const ast = j(source);
    ast
        .find(j.FunctionDeclaration)
        .forEach(({ value }) => {
            if (value.returnType === null)
                value.returnType = getReturnType(esTreeNodeToTSNode(value));
        });
    ast
        .find(j.ClassMethod, { kind: 'method' })
        .forEach(({ value }) => {
            if (value.returnType === null)
                value.returnType = getReturnType(esTreeNodeToTSNode(value).parent);
        });
    return ast.toSource();
}

export const parser = 'ts';
```

## Использование парсера typescript-eslint

Как было показано выше, хоть и отображение с использованием позиций узлов работает, оно не дает точного результата и иногда требует «ручной доводки». Более общим решением было бы написать явное отображение узлов ESTree в TSTree. Именно так работает парсер проекта [typescript-eslint](https://github.com/typescript-eslint/typescript-eslint). Воспользуемся им.

Для начала, нам нужно переопределить встроенный [парсер jscodeshift](https://github.com/facebook/jscodeshift#parser) на [парсер typescript-eslint](https://github.com/typescript-eslint/typescript-eslint/tree/master/packages/typescript-estree). В простейшем случае код выглядит так:

```javascript
export const parser = {
    parse(source) {
        return typescriptEstree.parse(source);
    }
};
```

Однако, нам придется немного усложнить код, чтобы получить отображение узлов ESTree в TSTree и `typeChecker`. Для этого в typescript-eslint используется функция `parseAndGenerateServices`. Чтобы все заработало, мы должны передать в нее путь к `.ts` файлу и путь к файлу конфигурации `tsconfig.json`. Так как прямого способа сделать этого нет, придется воспользоваться глобальной переменной (ох!):

```javascript
const parserState = {};

function parseWithServices(j, source, path, projectPath) {
    parserState.options = { filePath: path, project: projectPath };
    return {
        ast: j(source),
        services: parserState.services
    };
}

export const parser = {
    parse(source) {
        if (parserState.options !== undefined) {
            const options = parserState.options;
            delete parserState.options;
            const { ast, services } = typescriptEstree.parseAndGenerateServices(source, options);
            parserState.services = services;
            return ast;
        }
        return typescriptEstree.parse(source);
    }
};
```

Каждый раз, когда мы хотим получить расширенный набор средств парсера typescript-eslint, мы вызываем функцию `parseWithServices`, в которую передаем необходимые параметры (в остальных случаях мы по-прежнему используем функцию `j`):

```javascript
const { ast, services: { program, esTreeNodeToTSNodeMap } } = parseWithServices(j, source, path, tsConfigPath);

const typeChecker = program.getTypeChecker();
const esTreeNodeToTSNode = ({ original }) => esTreeNodeToTSNodeMap.get(original);
```

Остается только написать код обхода и модификации функций и методов классов:

```javascript
ast
    .find(j.FunctionDeclaration)
    .forEach(({ value }) => {
        if (value.returnType === null)
            value.returnType = getReturnType(esTreeNodeToTSNode(value));
    });
ast
    .find(j.MethodDefinition, { kind: 'method' })
    .forEach(({ value }) => {
        if (value.value.returnType === null)
            value.value.returnType = getReturnType(esTreeNodeToTSNode(value));
    });
return ast.toSource();
```

Надо отметить, что нам пришлось заменить селектор `ClassMethod` на `MethodDefinition`, чтобы обойти методы классов (также немного изменился код доступа к возвращаемому значению метода). Это специфика парсера typescript-eslint. Код функции `getReturnType` идентичен тому, что использовался ранее.

Полный листинг:

```javascript
import * as typescriptEstree from '@typescript-eslint/typescript-estree';

export default function transform({ source, path }, { j }, { tsConfigPath }) {
    const { ast, services: { program, esTreeNodeToTSNodeMap } } = parseWithServices(j, source, path, tsConfigPath);

    const typeChecker = program.getTypeChecker();
    const esTreeNodeToTSNode = ({ original }) => esTreeNodeToTSNodeMap.get(original);

    function getReturnTypeFromString(typeString) {
        let ret;
        j(`function foo(): ${typeString} { }`)
            .find(j.FunctionDeclaration)
            .some(({ value: { returnType } }) => ret = returnType);
        return ret;
    }

    function getReturnType(node) {
        return getReturnTypeFromString(
            typeChecker.typeToString(
                typeChecker.getReturnTypeOfSignature(
                    typeChecker.getSignatureFromDeclaration(node)
                )
            )
        );
    }

    ast
        .find(j.FunctionDeclaration)
        .forEach(({ value }) => {
            if (value.returnType === null)
                value.returnType = getReturnType(esTreeNodeToTSNode(value));
        });
    ast
        .find(j.MethodDefinition, { kind: 'method' })
        .forEach(({ value }) => {
            if (value.value.returnType === null)
                value.value.returnType = getReturnType(esTreeNodeToTSNode(value));
        });
    return ast.toSource();
}

const parserState = {};

function parseWithServices(j, source, path, projectPath) {
    parserState.options = { filePath: path, project: projectPath };
    return {
        ast: j(source),
        services: parserState.services
    };
}

export const parser = {
    parse(source) {
        if (parserState.options !== undefined) {
            const options = parserState.options;
            delete parserState.options;
            const { ast, services } = typescriptEstree.parseAndGenerateServices(source, options);
            parserState.services = services;
            return ast;
        }
        return typescriptEstree.parse(source);
    }
};
```

## Плюсы и минусы подходов

### Подход с номерами строк и столбцов

Плюсы:
 * Не требует переопределения встроенного парсера jscodeshift.
 * Гибкость передачи конфигурации и исходных текстов (можно передавать как файлы, так и строки/объекты в памяти, см. ниже).

Минусы:
 * Отображение узлов по позициям является неточным и в некоторых случаях требует корректировки.

### Подход с парсером typescript-eslint

Плюсы:
 * Точное отображение узлов из одного AST в другое.

Минусы:
 * Структура AST парсера typescript-eslint немного отличается от встроенного парсера jscodeshift.
 * Необходимость использовать файлы для передачи конфигурации TS и исходных текстов.

## Заключение

Первый подход легко добавить в существующие проекты, т.к. он не требует переопределения парсера, но отображение узлов AST, скорее всего, потребует корректировки.

Решение о втором подходе лучше принимать заранее, иначе, вероятно, придется тратить время на отладку кода из-за изменившейся структуры AST. С другой стороны, у вас будет полноценное отображение одних узлов на другие (и обратно).

## P.S.

Выше упоминалось, что при использовании парсера TS, можно передавать конфигурации и исходные тексты как в виде файлов, так и в виде объектов в памяти. Передача конфигурации в виде объекта и передача исходного текста в виде файла были рассмотрены в примере. Далее приводится код функций, которые позволяют прочитать конфигурацию из файла:

```javascript
class TsDiagnosticError extends Error {
    constructor(err) {
        super(Array.isArray(err) ? err.map(e => e.messageText).join('\n') : err.messageText);
        this.diagnostic = err;
    }
}

function tsGetCompilerOptionsFromConfigFile(tsConfigPath, basePath = '.') {
    const { config, error } = ts.readConfigFile(tsConfigPath, ts.sys.readFile);
    if (error)
        throw new TsDiagnosticError(error);
    const { options, errors } = ts.parseJsonConfigFileContent(config, tsGetCompilerOptionsFromConfigFile.host, basePath);
    if (errors.length !== 0)
        throw new TsDiagnosticError(errors);
    return options;
}

tsGetCompilerOptionsFromConfigFile.host = {
    fileExists: ts.sys.fileExists,
    readFile: ts.sys.readFile,
    readDirectory: ts.sys.readDirectory,
    useCaseSensitiveFileNames: true
};
```

И создать TS-программу из строки:

```javascript
function tsCreateStringSourceCompilerHost(mockPath, source, compilerOptions, setParentNodes) {
    const host = ts.createCompilerHost(compilerOptions, setParentNodes);

    const getSourceFileOriginal = host.getSourceFile.bind(host);
    const readFileOriginal = host.readFile.bind(host);
    const fileExistsOriginal = host.fileExists.bind(host);

    host.getSourceFile = (fileName, languageVersion, onError, shouldCreateNewSourceFile) => {
        return fileName === mockPath ?
            ts.createSourceFile(fileName, source, languageVersion) :
            getSourceFileOriginal(fileName, languageVersion, onError, shouldCreateNewSourceFile);
    };
    host.readFile = (fileName) => {
        return fileName === mockPath ?
            source :
            readFileOriginal(fileName);
    };
    host.fileExists = (fileName) => {
        return fileName === mockPath ?
            true :
            fileExistsOriginal(fileName);
    };

    return host;
}

function tsCreateStringSourceProgram(source, compilerOptions, mockPath = '_source.ts') {
    return ts.createProgram([mockPath], compilerOptions, tsCreateStringSourceCompilerHost(mockPath, source, compilerOptions));
}
```

## Ссылки

 * [jscodeshift](https://github.com/facebook/jscodeshift)
 * [TypeScript](https://github.com/microsoft/TypeScript)
 * [Using the Compiler API](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API)
 * [typescript-eslint](https://github.com/typescript-eslint/typescript-eslint)
 * [How to get `CompilerOptions` from `tsconfig.json`](https://stackoverflow.com/questions/53804566/how-to-get-compileroptions-from-tsconfig-json)
 * [How do I type check a snippet of TypeScript code in memory?](https://stackoverflow.com/questions/53733138/how-do-i-type-check-a-snippet-of-typescript-code-in-memory)
 * [Use compiler API for type inference](https://stackoverflow.com/questions/49355257/use-compiler-api-for-type-inference)
 * [how to use typescript Compiler API to get normal function info, eg: returnType/parameters?](https://stackoverflow.com/questions/47215069/how-to-use-typescript-compiler-api-to-get-normal-function-info-eg-returntype-p)
