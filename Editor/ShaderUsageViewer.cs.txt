using System.Collections.Generic;
using System.IO;
using UnityEditor;
using UnityEngine;

namespace Game.Util.Editor
{
    /// <summary>
    /// insturction count and precision value are subjected to potential fallacies depending on variable names in a shader.
    /// todo should improve the precision of substring detection via more delicate constraints later.
    /// </summary>
    public class ShaderUsageViewer : EditorWindow
    {
        private ShaderModel selectedShaderModel = ShaderModel.Unknown;
        private Dictionary<Shader, ShaderInfo> shaderUsage = new Dictionary<Shader, ShaderInfo>();
        private Vector2 scrollPos;
        
        [MenuItem("Tools/Shader Usage Viewer")]
        public static void ShowWindow()
        {
            GetWindow<ShaderUsageViewer>("Shader Usage Viewer");
        }

        private void OnGUI()
        {
            if (GUILayout.Button("Scan Project for Shaders"))
            {
                ScanProject();
            }
            
            EditorGUILayout.LabelField("Unique Shader Count : " + shaderUsage.Count);
            GUILayout.Space(10);
            
            if (shaderUsage.Count == 0)
            {
                EditorGUILayout.HelpBox("No shaders found. Please click the button to scan the project.", MessageType.Info);
            }
            else
            {
                scrollPos = EditorGUILayout.BeginScrollView(scrollPos);

                foreach (var entry in shaderUsage)
                {
                    GUI.color = Color.white;
                    EditorGUILayout.BeginVertical("box");
                    EditorGUILayout.LabelField("Shader: " + entry.Key.name, EditorStyles.boldLabel);
                    SetGUIColorForShaderModel(entry.Value.shaderModel);
                    GUILayout.Space(5);
                    EditorGUILayout.BeginHorizontal(GUILayout.ExpandWidth(true)); // Adjust height as needed
                    EditorGUILayout.LabelField("Shader Model: " + entry.Value.shaderModel);
                    GUI.color = entry.Value.precision == ShaderPrecision.Float ? Color.red : Color.white;
                    GUILayout.Space(-35);
                    EditorGUILayout.LabelField("Precision: " + entry.Value.precision.ToString());
                    GUI.color = entry.Value.instructionCount > 450 ? Color.red : Color.white;
                    GUILayout.Space(-55);
                    EditorGUILayout.LabelField("Instruction Count: " + entry.Value.instructionCount.ToString());
                    GUI.color = Color.white;
                    GUILayout.Space(-55);
                    EditorGUILayout.LabelField("Texture Samples: " + entry.Value.textureSampleCount.ToString());
                    EditorGUILayout.EndHorizontal();
                    EditorGUILayout.LabelField("Materials Using This Shader:");

                    GUILayout.Space(5);
                    entry.Value.isExpanded = EditorGUILayout.Foldout(entry.Value.isExpanded, "Materials Using This Shader");

                    if (entry.Value.isExpanded)
                    {
                        EditorGUI.indentLevel++;
                        foreach (var material in entry.Value.materials)
                        {
                            EditorGUILayout.ObjectField(material, typeof(Material), false);
                        }
                        
                        EditorGUI.indentLevel--;
                    }


                    EditorGUILayout.EndVertical();
                }
                
                EditorGUILayout.Space();
        
                EditorGUILayout.LabelField("Select Shader Model to Filter Shaders", EditorStyles.boldLabel);
                selectedShaderModel = (ShaderModel) EditorGUILayout.EnumPopup("Shader Model", selectedShaderModel);
            
                if (GUILayout.Button("Show Shaders Using Selected Model"))
                {
                    ShowShadersUsingSelectedModel();
                }

                EditorGUILayout.EndScrollView();
            }
        }

        private void ScanProject()
        {
            shaderUsage.Clear();

            string[] materialGuids = AssetDatabase.FindAssets("t:Material");
            foreach (string guid in materialGuids)
            {
                string path = AssetDatabase.GUIDToAssetPath(guid);
                Material material = AssetDatabase.LoadAssetAtPath<Material>(path);

                if (material != null)
                {
                    Shader shader = material.shader;

                    if (shader != null && !IsBuiltinResource(shader))
                    {
                        if (!shaderUsage.ContainsKey(shader))
                        {
                            shaderUsage[shader] = new ShaderInfo
                            {
                                shaderModel = GetShaderModel(shader),
                                precision = GetShaderPrecision(shader),
                                instructionCount = GetInstructionCount(shader),
                                textureSampleCount = GetTextureSampleCount(shader),
                                materials = new List<Material>()
                            };
                        }

                        shaderUsage[shader].materials.Add(material);
                    }
                }
            }

            Debug.Log("Shader scan completed. Found " + shaderUsage.Count + " shaders.");
        }
        
        private bool IsBuiltinResource(Object obj)
        {
            return AssetDatabase.GetAssetPath(obj).StartsWith("Resources/unity_builtin_extra");
        }

        private ShaderModel GetShaderModel(Shader shader)
        {
            string shaderPath = AssetDatabase.GetAssetPath(shader);
            
            if (!File.Exists(shaderPath))
                return ShaderModel.Unknown;
            
            string shaderCode = File.ReadAllText(shaderPath);

            if (shaderCode.Contains("target 2.0")) return ShaderModel._2_0;
            if (shaderCode.Contains("target 2.5")) return ShaderModel._2_5;
            if (shaderCode.Contains("target 3.0")) return ShaderModel._3_0;
            if (shaderCode.Contains("target 3.5")) return ShaderModel._3_5;
            if (shaderCode.Contains("target 4.0")) return ShaderModel._4_0;
            if (shaderCode.Contains("target 4.5")) return ShaderModel._4_5;
            if (shaderCode.Contains("target 5.0")) return ShaderModel._5_0;
            if (shaderCode.Contains("target 5.1")) return ShaderModel._5_1;
            if (shaderCode.Contains("target 6.0")) return ShaderModel._6_0;
            if (shaderCode.Contains("target 6.2")) return ShaderModel._6_2;

            return ShaderModel.Unknown;
        }
        
        private void ShowShadersUsingSelectedModel()
        {
            List<Shader> shaders = new List<Shader>();

            foreach (var entry in shaderUsage)
            {
                if (entry.Value.shaderModel == selectedShaderModel)
                {
                    shaders.Add(entry.Key);
                }
            }

            if (shaders.Count > 0)
            {
                ShaderListPopup window = CreateInstance<ShaderListPopup>();
                window.Init(shaders);
                window.ShowUtility();
            }
            else
            {
                EditorUtility.DisplayDialog("No Shaders Found", "No shaders found using the selected shader model.", "OK");
            }
        }

        
        public enum ShaderModel
        {
            Unknown,
            _2_0,
            _2_5,
            _3_0,
            _3_5,
            _4_0,
            _4_5,
            _5_0,
            _5_1,
            _6_0,
            _6_2
        }
        
        private enum ShaderPrecision
        {
            Unknown,
            Half,
            Float,
            Fixed
        }
    
        private class ShaderInfo
        {
            public ShaderModel shaderModel;
            public ShaderPrecision precision;
            public int instructionCount;
            public bool isExpanded = false; 
            public int textureSampleCount;
            public List<Material> materials;
        }
        
        private class ShaderListPopup : EditorWindow
        {
            private List<Shader> shaders;
            private Vector2 scrollPos;

            public void Init(List<Shader> shaders)
            {
                this.shaders = shaders;
                titleContent = new GUIContent("Shaders Using Selected Model");
            }

            private void OnGUI()
            {
                scrollPos = EditorGUILayout.BeginScrollView(scrollPos);

                foreach (var shader in shaders)
                {
                    EditorGUILayout.ObjectField(shader, typeof(Shader), false);
                }

                EditorGUILayout.EndScrollView();
            }
        }
        
        //todo ensure realiability of the method // --> if a variable contains any of the following substrings those would as well be counted. may result in slightly inflated numbers.
        private int GetInstructionCount(Shader shader)
        {
            string shaderPath = AssetDatabase.GetAssetPath(shader);

            if (!File.Exists(shaderPath))
                return 0;

            string shaderCode = File.ReadAllText(shaderPath);
            int instructionCount = 0;

            string[] operations = {"add", "mul", "sub", "div", "dot", "cross", "normalize", "lerp", "sin", "cos", "tan", "exp", "log"};

            foreach (string op in operations)
            {
                instructionCount += CountOccurrences(shaderCode, op);
            }

            return instructionCount;
        }
        
        private int CountOccurrences(string source, string substring)
        {
            int count = 0;
            int index = 0;

            while ((index = source.IndexOf(substring, index)) != -1)
            {
                index += substring.Length;
                count++;
            }

            return count;
        }
        
        //todo also same here if any variable contains any of the following substrings, we may detect a wrong precision value.
        //we shall check the shader itself to ensure such cases.
        private ShaderPrecision GetShaderPrecision(Shader shader)
        {
            string shaderPath = AssetDatabase.GetAssetPath(shader);

            if (!File.Exists(shaderPath))
                return ShaderPrecision.Unknown;

            string shaderCode = File.ReadAllText(shaderPath);

            if (shaderCode.Contains("half")) return ShaderPrecision.Half;
            if (shaderCode.Contains("float")) return ShaderPrecision.Float;
            if (shaderCode.Contains("fixed")) return ShaderPrecision.Fixed;

            return ShaderPrecision.Unknown;
        }
        
        private int GetTextureSampleCount(Shader shader)
        {
            string shaderPath = AssetDatabase.GetAssetPath(shader);

            if (!File.Exists(shaderPath))
                return 0;

            string shaderCode = File.ReadAllText(shaderPath);

            int textureSampleCount = CountOccurrences(shaderCode, "tex2D("); // tex2D( syntax can only appear in frag funtion thus we ignore property declerations or property names that contain tex2D

            return textureSampleCount;
        }
        
        private Color SetGUIColorForShaderModel(ShaderModel shaderModel)
        {
            switch (shaderModel)
            {
                case ShaderModel._2_0:
                    GUI.color = Color.green;
                    break;
                case ShaderModel._3_0:
                    GUI.color = Color.green;
                    break;
                case ShaderModel._3_5:
                    GUI.color = Color.green;
                    break;
                case ShaderModel._4_0:
                    GUI.color = Color.red;
                    break;
                case ShaderModel._4_5:
                    GUI.color = Color.red;
                    break;
                case ShaderModel._5_0:
                    GUI.color = Color.red;
                    break;
                case ShaderModel._5_1:
                    GUI.color = Color.red;
                    break;
                case ShaderModel._6_0:
                    GUI.color = Color.red;
                    break;
                case ShaderModel._6_2:
                    GUI.color = Color.red;
                    break;
                default:
                    GUI.color = Color.white;
                    break;
            }
            
            return GUI.color;
        }
    }
}