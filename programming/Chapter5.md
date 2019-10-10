## Example 5: Thermal Dynamic Analysis Example

The present example demonstrates the solution of a dynamic thermal flux problem.

This example consists of three parts. The first one has to do with the definition of the model,
the second one with the solution of the model and the third one compares the generated results
to the expected ones:

```csharp
private static void RunTest()
{
	Model model = CreateModel();
	IVectorView solution = SolveModel(model);
	Assert.True(CompareResults(solution));
}
```

In the first part, the model and the subdomain are created:
```csharp
var model = new Model();

// Subdomains
model.SubdomainsDictionary.Add(0, new Subdomain(subdomainID));
```

as long as the material:
```csharp
// Material
double density = 1.0;
double k = 1.0;
double c = 1.0;
```

and the nodal coordinates:
```csharp
// Nodes
int numNodes = 9;
var nodes = new Node[numNodes];
nodes[0] = new Node(id: 0, x: 0.0, y: 0.0);
nodes[1] = new Node(id: 1, x: 1.0, y: 0.0);
nodes[2] = new Node(id: 2, x: 2.0, y: 0.0);
nodes[3] = new Node(id: 3, x: 0.0, y: 1.0);
nodes[4] = new Node(id: 4, x: 1.0, y: 1.0);
nodes[5] = new Node(id: 5, x: 2.0, y: 1.0);
nodes[6] = new Node(id: 6, x: 0.0, y: 2.0);
nodes[7] = new Node(id: 7, x: 1.0, y: 2.0);
nodes[8] = new Node(id: 8, x: 2.0, y: 2.0);

for (int i = 0; i < numNodes; ++i)
{
	model.NodesDictionary[i] = nodes[i];
}
```

The elements are defined according to the connectivity matrix and then are added to the model 
and the subdomain defined earlier:
```csharp
// Elements
int numElements = 4;
var elementFactory = new ThermalElement2DFactory(1.0, new ThermalMaterial(density, c, k));
var elements = new ThermalElement2D[4];
elements[0] = elementFactory.CreateElement(CellType.Quad4, new Node[] { nodes[0], nodes[1], nodes[4], nodes[3] });
elements[1] = elementFactory.CreateElement(CellType.Quad4, new Node[] { nodes[1], nodes[2], nodes[5], nodes[4] });
elements[2] = elementFactory.CreateElement(CellType.Quad4, new Node[] { nodes[3], nodes[4], nodes[7], nodes[6] });
elements[3] = elementFactory.CreateElement(CellType.Quad4, new Node[] { nodes[4], nodes[5], nodes[8], nodes[7] });

for (int i = 0; i < numElements; ++i)
{
	var elementWrapper = new Element() { ID = i, ElementType = elements[i] };
	foreach (var node in elements[i].Nodes)
	{
		elementWrapper.AddNode(node);
	}

	model.ElementsDictionary[i] = elementWrapper;
	model.SubdomainsDictionary[subdomainID].Elements.Add(elementWrapper);
}
```

And finally, the Dirichlet and Neumann boundary conditions are defined as:
```csharp
// Dirichlet BC
model.NodesDictionary[0].Constraints.Add(new Constraint() { DOF = ThermalDof.Temperature, Amount = 100.0 });
model.NodesDictionary[3].Constraints.Add(new Constraint() { DOF = ThermalDof.Temperature, Amount = 100.0 });
model.NodesDictionary[6].Constraints.Add(new Constraint() { DOF = ThermalDof.Temperature, Amount = 100.0 });

// Neumann BC
double q = 50.0;
model.Loads.Add(new Load() { Amount = q / 2.0, Node = model.NodesDictionary[2], DOF = ThermalDof.Temperature });
model.Loads.Add(new Load() { Amount = q, Node = model.NodesDictionary[5], DOF = ThermalDof.Temperature });
model.Loads.Add(new Load() { Amount = q / 2.0, Node = model.NodesDictionary[8], DOF = ThermalDof.Temperature });

return model;
```

In the second part, we define a Skyline solver and a Thermal problem provider, along
with a Linear child analyzer and a Thermal Dynamic analyzer as a parent analyzer, 
according to the following:
```csharp
SkylineSolver solver = (new SkylineSolver.Builder()).BuildSolver(model);
var provider = new ProblemThermal(model, solver);

var childAnalyzer = new LinearAnalyzer(model, solver, provider);
var parentAnalyzer = new ThermalDynamicAnalyzer(model, solver, provider, childAnalyzer, 0.5, 0.5, 1000);

parentAnalyzer.Initialize();
parentAnalyzer.Solve();

return solver.LinearSystems[subdomainID].Solution;
```

The third and last part of the example compares the solution of the example with an expected value:
```csharp
private static bool CompareResults(IVectorView solution)
{
	var comparer = new ValueComparer(1E-5);
	var expectedSolution = Vector.CreateFromArray(new double[] { 150, 200, 150, 200, 150, 200 });
	int numFreeDofs = 6;
	if (solution.Length != 6)
	{
		return false;
	}
	for (int i = 0; i < numFreeDofs; ++i)
	{
		if (!comparer.AreEqual(expectedSolution[i], solution[i]))
		{
			return false;
		}
	}
	return true;
}
```