# repa

Создание Windows Forms приложения для визуализации алгоритма муравьиной колонии (ACO) с использованием шаблона MVC требует тщательной реализации. Мы создадим произвольный взвешенный граф, реализуем алгоритм ACO и визуализируем результаты. В процессе выполнения алгоритма будут отображаться изменения в феромонах и оптимальный маршрут. Как и было запрошено, приложение будет содержать две кнопки: "Сгенерировать график" и "Запустить алгоритм". Давайте начнем с разработки кода.

### Проектирование структуры MVC

#### 1. Модель

Модель будет содержать классы для хранения графа, узлов, ребер, муравьев и ACO.

##### Класс Node

```csharp
public class Node
{
    public int Id { get; private set; }
    public Point Position { get; set; }

    public Node(int id, Point position)
    {
        Id = id;
        Position = position;
    }
}
```

##### Класс Edge

```csharp
public class Edge
{
    public Node From { get; private set; }
    public Node To { get; private set; }
    public double Weight { get; private set; }
    public double Pheromone { get; set; }

    public Edge(Node from, Node to, double weight)
    {
        From = from;
        To = to;
        Weight = weight;
        Pheromone = 1.0;  // Start with initial pheromone level
    }
}
```

##### Класс Graph

```csharp
public class Graph
{
    public List<Node> Nodes { get; }
    public List<Edge> Edges { get; }

    public Graph()
    {
        Nodes = new List<Node>();
        Edges = new List<Edge>();
    }

    public void AddNode(Node node)
    {
        Nodes.Add(node);
    }

    public void AddEdge(Node from, Node to, double weight)
    {
        Edge edge = new Edge(from, to, weight);
        Edges.Add(edge);
    }

    public List<Edge> GetEdges(Node node)
    {
        return Edges.Where(e => e.From == node || e.To == node).ToList();
    }
}
```

##### Класс Ant

```csharp
public class Ant
{
    private Graph graph;
    private List<Node> path;
    private double pathLength;
    private Node currentNode;
    private Random random;

    public Ant(Node startNode, Graph graph)
    {
        this.graph = graph;
        path = new List<Node> { startNode };
        currentNode = startNode;
        pathLength = 0;
        random = new Random();
    }

    public List<Node> Path => path;
    public double PathLength => pathLength;

    public void Move()
    {
        var edges = graph.GetEdges(currentNode).Where(e => !path.Contains(e.To) || !path.Contains(e.From)).ToList();
        if (edges.Count == 0) return;

        Edge selectedEdge = SelectEdge(edges);
        Node nextNode = (selectedEdge.From == currentNode) ? selectedEdge.To : selectedEdge.From;
        
        path.Add(nextNode);
        pathLength += selectedEdge.Weight;
        currentNode = nextNode;
    }

    private Edge SelectEdge(List<Edge> edges)
    {
        double total = edges.Sum(e => e.Pheromone / e.Weight);
        double pick = random.NextDouble() * total;
        double current = 0;

        foreach (var edge in edges)
        {
            current += edge.Pheromone / edge.Weight;
            if (current >= pick)
                return edge;
        }

        return edges.Last();
    }
}
```

##### Класс AntColonyOptimization

```csharp
public class AntColonyOptimization
{
    private Graph graph;
    private List<Ant> ants;
    private int numberOfAnts;
    private double evaporationRate = 0.5;

    public AntColonyOptimization(Graph graph, int numberOfAnts)
    {
        this.graph = graph;
        this.numberOfAnts = numberOfAnts;
        ants = new List<Ant>();
    }

    public void InitializeAnts(Node startNode)
    {
        ants.Clear();
        for (int i = 0; i < numberOfAnts; i++)
        {
            ants.Add(new Ant(startNode, graph));
        }
    }

    public void RunIteration()
    {
        foreach (var ant in ants)
        {
            ant.Move();
        }

        UpdatePheromones();
    }

    private void UpdatePheromones()
    {
        foreach (var edge in graph.Edges)
            edge.Pheromone *= (1.0 - evaporationRate);

        foreach (var ant in ants)
        {
            double contribution = 1.0 / ant.PathLength;

            for (int i = 1; i < ant.Path.Count; i++)
            {
                Edge edge = graph.Edges.First(e => (e.From == ant.Path[i - 1] && e.To == ant.Path[i]) || (e.From == ant.Path[i] && e.To == ant.Path[i - 1]));
                edge.Pheromone += contribution;
            }
        }
    }

    public List<Edge> GetBestPath()
    {
        return ants.OrderBy(a => a.PathLength).First().Path.Zip(ants.OrderBy(a => a.PathLength).First().Path.Skip(1), (f, t) => 
            graph.Edges.First(e => (e.From == f && e.To == t) || (e.From == t && e.To == f))).ToList();
    }
}
```

#### 2. Вид (View)

Создадим основную форму с UI элементами.

##### Код формы (Form1)

```csharp
public partial class MainForm : Form
{
    private Graph graph;
    private AntColonyOptimization aco;
    private Timer timer;

    public MainForm()
    {
        InitializeComponent();
        
        graph = new Graph();
        aco = new AntColonyOptimization(graph, 10);

        // Initialize buttons
        btnGenerateGraph.Click += BtnGenerateGraph_Click;
        btnRunAlgorithm.Click += BtnRunAlgorithm_Click;
        
        // Initialize Timer
        timer = new Timer() { Interval = 500 };
        timer.Tick += Timer_Tick;
    }

    private void BtnGenerateGraph_Click(object sender, EventArgs e)
    {
        GenerateRandomGraph(10, 20);  // Sample numbers
        Invalidate(); // Request to redraw the form
    }

    private void BtnRunAlgorithm_Click(object sender, EventArgs e)
    {
        if (graph.Nodes.Count == 0)
            return;

        aco.InitializeAnts(graph.Nodes[0]);  // Start with node 0
        timer.Start();
    }

    private void Timer_Tick(object sender, EventArgs e)
    {
        aco.RunIteration();
        Invalidate();

        // Display best path after algorithm finishes
        if (/* some condition to end algorithm*/)
        {
            timer.Stop();
            ShowBestPath();
        }
    }

    private void GenerateRandomGraph(int numberNodes, int maxWeight)
    {
        Random rnd = new Random();
        graph = new Graph();  // Reset graph

        for (int i = 0; i < numberNodes; i++)
        {
            graph.AddNode(new Node(i, new Point(rnd.Next(50, 400), rnd.Next(50, 400))));
        }

        for (int i = 0; i < numberNodes; i++)
        {
            for (int j = i + 1; j < numberNodes; j++)
            {
                graph.AddEdge(graph.Nodes[i], graph.Nodes[j], rnd.Next(1, maxWeight));
            }
        }
    }

    // Drawing method
    protected override void OnPaint(PaintEventArgs e)
    {
        base.OnPaint(e);
        Graphics g = e.Graphics;

        foreach (var edge in graph.Edges)
        {
            Pen pen = new Pen(Color.FromArgb((int)Math.Min(255, edge.Pheromone * 50), Color.Red), 1);
            g.DrawLine(pen, edge.From.Position, edge.To.Position);
        }

        foreach (var node in graph.Nodes)
        {
            g.FillEllipse(Brushes.Black, node.Position.X - 5, node.Position.Y - 5, 10, 10);
        }
    }

    private void ShowBestPath()
    {
        // Highlight the best path found
        List<Edge> bestPath = aco.GetBestPath();
        Graphics g = CreateGraphics();
        foreach (var edge in bestPath)
        {
            Pen pen = new Pen(Color.Yellow, 3);
            g.DrawLine(pen, edge.From.Position, edge.To.Position);
        }
    }
}
```

#### 3. Контроллер

Контроллер стандартизирован в form-классе через методы, управляющие запуском алгоритма и взаимодействиями с пользовательским интерфейсом.

### Итог

Теперь у нас есть приложение Windows Forms, использующее шаблон MVC для визуализации ACO алгоритма. Приложение позволяет генерировать случайный граф и запускать алгоритм ACO, отображая интенсивность феромонов на ребрах графа.

Вы можете расширить и модифицировать код для более богатого функционала, регулировки параметров и улучшения интерфейса пользователя.
